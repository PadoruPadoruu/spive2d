<script>
  import { onMount } from 'svelte';
  import { appState, sceneDir } from '#lib/appState.svelte.js';
  import { getRenderer, setRenderer } from '#lib/rendererStore.svelte.js';
  import { createRenderer, getRendererType } from '#lib/renderer/createRenderer.js';
  import { preloadManager } from '#lib/renderer/preloadManager.js';
  import { filterSceneIndices, findIdleAnimation, sanitizeInputUrl, sanitizeFilename } from '#lib/utils.js';
  import { getAssetUrl, getExportDirectory } from '#lib/fileManager.js';
  import { exportImage, exportAnimation, exportImageSequence } from '#lib/exporter.js';
  import { exportModelFiles } from '#lib/modelExporter.js';
  import { createTransformAction, isViewerPoint } from '#lib/inputAction.js';
  import { loadSetting, saveSetting } from '#lib/settings.js';
  import { showNotification } from '#lib/notificationStore.svelte.js';
  import { t } from '#lib/i18n.svelte.js';
  import { getShortcuts } from '#lib/shortcutKeys.js';
  import SettingsDialog from './SettingsDialog.svelte';
  import Sidebar from './Sidebar.svelte';
  import AnimationController from './AnimationController.svelte';
  import Notification from './Notification.svelte';
  import ExportQueue from './ExportQueue.svelte';
  import SceneThumbnailGrid from './SceneThumbnailGrid.svelte';
  import PickOverlay from './PickOverlay.svelte';
  import HitAreaOverlay from './HitAreaOverlay.svelte';
  import InteractionCoordinateOverlay from './InteractionCoordinateOverlay.svelte';
  import { thumbnailManager } from '#lib/thumbnailManager.svelte.js';
  import { invoke, convertFileSrc } from '@tauri-apps/api/core';
  import { listen } from '@tauri-apps/api/event';
  import { join } from '@tauri-apps/api/path';
  import { mkdir } from '@tauri-apps/plugin-fs';
  import { getCurrentWindow } from '@tauri-apps/api/window';
  import { toggleFullscreen, exitFullscreen } from '#lib/windowManager.js';
  import { openExportDirectory } from '#lib/fileManager.js';

  if (typeof window !== 'undefined') {
    window.__TAURI__ = window.__TAURI__ || {};
    window.__TAURI__.core = window.__TAURI__.core || {};
    window.__TAURI__.core.convertFileSrc = convertFileSrc;
  }

  let dialogOpen = $state(true);
  let sceneGridOpen = $state(false);
  let showSpinner = $state(false);
  let canvasContainer = $state();
  let sidebar = $state();
  let settingsDialog = $state();
  let animController = $state();
  let shortcuts = $state(getShortcuts());
  let interactionRestoreAnimation = '';
  let interactionStart = null;
  let interactionCoordinate = $state(null);
  let coordinateProbePointerId = null;
  const transformAction = createTransformAction();
  let currentLoadId = 0;
  let loadingRenderers = [];

  function refreshShortcuts() {
    shortcuts = getShortcuts();
  }

  function isCoordinateProbePoint(event) {
    if (!appState.showInteractionCoordinates || event.button !== 0 ||
      !appState.initialized || appState.processing || dialogOpen || sceneGridOpen ||
      !isViewerPoint(event.clientX, event.clientY) ||
      event.clientX >= window.innerWidth - 250 ||
      event.target?.closest?.('button, input, select, textarea, dialog, #sidebar')) return false;
    const renderer = getRenderer();
    const canvas = renderer?.getCanvas?.();
    if (!renderer || !canvas) return false;
    const rect = canvas.getBoundingClientRect();
    if (event.clientX < rect.left || event.clientX > rect.right ||
      event.clientY < rect.top || event.clientY > rect.bottom) return false;
    return true;
  }

  function getInteractionCoordinateRecord(renderer, clientX, clientY, includeTarget = true) {
    const canvas = renderer.getCanvas();
    const rect = canvas.getBoundingClientRect();
    const target = includeTarget ? renderer.getInteractionTargetAt?.(clientX, clientY) ?? '' : '';
    const coordinates = renderer.getInteractionCoordinatesAt?.(clientX, clientY) ?? {
      canvasX: clientX - rect.left,
      canvasY: clientY - rect.top,
      coordinateSpace: 'Canvas',
    };
    return { target, coordinates };
  }

  function updateInteractionCoordinate(renderer, clientX, clientY, record = null, includeTarget = true) {
    const { target, coordinates } = record || getInteractionCoordinateRecord(renderer, clientX, clientY, includeTarget);
    interactionCoordinate = { clientX, clientY, coordinates };
    return { target, coordinates };
  }

  function handleCoordinateProbePointerDown(event) {
    if (!isCoordinateProbePoint(event)) return;
    const renderer = getRenderer();
    const { target, coordinates } = updateInteractionCoordinate(renderer, event.clientX, event.clientY);
    coordinateProbePointerId = event.pointerId;
    console.info('[Spive2D Interaction Coordinates]', {
      renderer: renderer.constructor?.name || 'Unknown',
      target: target || null,
      screen: { x: event.clientX, y: event.clientY },
      canvas: { x: coordinates.canvasX, y: coordinates.canvasY },
      model: Number.isFinite(coordinates.modelX) && Number.isFinite(coordinates.modelY)
        ? { space: coordinates.coordinateSpace, x: coordinates.modelX, y: coordinates.modelY }
        : null,
    });
  }

  function handleCoordinateProbePointerMove(event) {
    if (!appState.showInteractionCoordinates || event.pointerId !== coordinateProbePointerId) return;
    const renderer = getRenderer();
    if (!renderer) return;
    const rect = renderer.getCanvas()?.getBoundingClientRect?.();
    if (!rect || event.clientX < rect.left || event.clientX > rect.right ||
      event.clientY < rect.top || event.clientY > rect.bottom) return;
    updateInteractionCoordinate(renderer, event.clientX, event.clientY, null, false);
  }

  function handleCoordinateProbePointerUp(event) {
    if (event.pointerId === coordinateProbePointerId) coordinateProbePointerId = null;
  }

  onMount(() => {
    initBackground();
    window.addEventListener('pointerdown', handleCoordinateProbePointerDown, true);
    window.addEventListener('pointermove', handleCoordinateProbePointerMove, true);
    window.addEventListener('pointerup', handleCoordinateProbePointerUp, true);
    window.addEventListener('pointercancel', handleCoordinateProbePointerUp, true);
    if (typeof window !== 'undefined') {
      const params = new URLSearchParams(window.location.search);
      const modelUrl = params.get('model');
      if (modelUrl) {
        processPath([modelUrl]);
      }
      window.__APP_STATE__ = appState;
      window.__GET_RENDERER__ = getRenderer;
    }
    const unlistenProgress = listen('progress', (event) => {
      appState.processing = event.payload;
      showSpinner = event.payload;
      if (showSpinner) {
        dialogOpen = false;
      }
    });
    const unlistenDragDrop = listen('tauri://drag-drop', async (event) => {
      processPath(event.payload.paths);
    });
    return async () => {
      window.removeEventListener('pointerdown', handleCoordinateProbePointerDown, true);
      window.removeEventListener('pointermove', handleCoordinateProbePointerMove, true);
      window.removeEventListener('pointerup', handleCoordinateProbePointerUp, true);
      window.removeEventListener('pointercancel', handleCoordinateProbePointerUp, true);
      (await unlistenProgress)();
      (await unlistenDragDrop)();
    };
  });

  $effect(() => {
    const renderer = getRenderer();
    if (renderer && appState.initialized) {
      renderer.applyTransform(
        appState.transform.scale,
        appState.transform.moveX,
        appState.transform.moveY,
        appState.transform.rotate
      );
    }
  });

  function initBackground() {
    const savedImagePath = loadSetting('spive2d_bg_image_path', '');
    const savedColor = loadSetting('spive2d_bg_color', '');
    if (savedImagePath) {
      document.body.style.backgroundColor = '';
      document.body.style.backgroundImage = `url("${getAssetUrl(savedImagePath)}")`;
      document.body.style.backgroundSize = 'cover';
      document.body.style.backgroundPosition = 'center';
    } else if (savedColor) {
      document.body.style.backgroundColor = savedColor;
      document.body.style.backgroundImage = 'none';
    }
  }

  async function processPath(paths) {
    if (appState.processing || paths.length === 0) return;
    paths = paths.map(sanitizeInputUrl);
    const wasInitialized = appState.initialized;
    appState.initialized = false;
    try {
      const inputPath = paths[0];
      let dirFiles = {};
      if (inputPath.startsWith('https://')) {
        let unityRes = null;
        let shouldInvokeBackend = false;
        try {
          showSpinner = true;
          let hasArchive = false;
          for (const p of paths) {
            try {
              const url = new URL(p, window.location.origin);
              const pathname = url.pathname.toLowerCase();
              if (pathname.endsWith('.zip') || pathname.endsWith('.7z')) {
                hasArchive = true;
                break;
              }
            } catch {}
          }
          const isUnity = !hasArchive && await invoke('probe_unity_url', { url: inputPath });
          if (isUnity && appState.skipUnity) {
            throw new Error('Unsupported file type');
          }
          shouldInvokeBackend = isUnity || hasArchive;
          if (shouldInvokeBackend) {
            unityRes = await invoke('handle_urls', { urls: paths, mergeSequential: appState.mergeSequential, skipUnity: appState.skipUnity });
          }
        } catch (e) {
          console.error(e);
          showNotification(e.message || String(e), 'error');
          appState.initialized = wasInitialized;
          return;
        } finally {
          showSpinner = false;
        }
        if (shouldInvokeBackend) {
          if (unityRes && Object.keys(unityRes).length > 0) {
            dirFiles = unityRes;
          } else {
            showNotification(t('noFilesFound'));
            appState.initialized = wasInitialized;
            return;
          }
        } else {
          const urlScenes = [];
          for (const path of paths) {
            let url;
            try {
              url = new URL(path, window.location.origin);
            } catch {
              continue;
            }
            const urlString = url.toString();
            const lastSlashIndex = urlString.lastIndexOf('/');
            const dirName = urlString.substring(0, lastSlashIndex + 1);
            const fileNameWithExt = urlString.substring(lastSlashIndex + 1);
            let baseName = fileNameWithExt;
            let ext1 = '';
            let ext2 = '';
            if (fileNameWithExt.endsWith('.model3.json')) {
              baseName = fileNameWithExt.substring(0, fileNameWithExt.length - '.model3.json'.length);
              ext1 = '.model3.json';
              ext2 = '.moc3';
            } else if (fileNameWithExt === 'model3.json') {
              baseName = 'model3';
              ext1 = '.model3.json';
              ext2 = '';
            } else if (fileNameWithExt.endsWith('.model.json') || fileNameWithExt === 'model.json') {
              baseName = fileNameWithExt === 'model.json' ? 'model' : fileNameWithExt.substring(0, fileNameWithExt.length - '.model.json'.length);
              ext1 = '.model.json';
              ext2 = '';
            } else if (fileNameWithExt.endsWith('.meta.json')) {
              baseName = fileNameWithExt.substring(0, fileNameWithExt.length - '.meta.json'.length);
              ext1 = '.meta.json';
              ext2 = '';
            } else if (fileNameWithExt.endsWith('.skel')) {
              baseName = fileNameWithExt.substring(0, fileNameWithExt.length - '.skel'.length);
              ext1 = '.skel';
              ext2 = '.atlas';
            } else if (fileNameWithExt.endsWith('.json')) {
              baseName = fileNameWithExt.substring(0, fileNameWithExt.length - '.json'.length);
              ext1 = '.json';
              ext2 = '.atlas';
            } else if (fileNameWithExt.endsWith('.asset')) {
              baseName = fileNameWithExt.substring(0, fileNameWithExt.length - '.asset'.length);
              ext1 = '.asset';
              ext2 = '.atlas';
            } else {
              continue;
            }
            urlScenes.push({ dirName, scene: { name: baseName, mainExt: ext1, atlasExt: ext2, files: [], isMerged: false } });
          }
          if (urlScenes.length === 0) {
            appState.initialized = wasInitialized;
            showNotification(t('invalidUrl'));
            return;
          }
          let rootDir = urlScenes[0].dirName;
          for (const { dirName } of urlScenes) {
            while (!dirName.startsWith(rootDir)) {
              rootDir = rootDir.substring(0, rootDir.lastIndexOf('/', rootDir.length - 2) + 1);
            }
          }
          const scenes = [];
          for (const { dirName, scene } of urlScenes) {
            scene.name = dirName.substring(rootDir.length) + scene.name;
            if (!scenes.some(item => item.name === scene.name)) scenes.push(scene);
          }
          dirFiles = { [rootDir]: scenes };
        }
      } else {
        dirFiles = await invoke('handle_dropped_paths', { paths, mergeSequential: appState.mergeSequential, skipUnity: appState.skipUnity, preferArchives: appState.preferArchives });
      }
      const rootDir = Object.keys(dirFiles)[0];
      if (!rootDir) {
        appState.initialized = wasInitialized;
        showNotification(t('noFilesFound'));
        return;
      }
      sceneGridOpen = false;
      thumbnailManager.clear();
      appState.directories = {
        files: dirFiles,
        selectedDir: rootDir,
        selectedScene: 0,
        sceneFilter: '',
      };
      const previousSkins = getRenderer()?.getPropertyItems?.('skins')?.filter(item => item.checked).map(item => item.name) || [];
      disposeModel();
      await initModel(previousSkins, { detectAlpha: true });
      appState.initialized = true;
      dialogOpen = false;
    } catch (error) {
      console.error('Error handling dropped path:', error);
      appState.initialized = wasInitialized;
      const errMsg = typeof error === 'string' ? error : (error?.message || String(error));
      if (errMsg.startsWith('HTTP ')) {
        showNotification(t('resourceNotFound'));
      } else if (errMsg.includes('Unsupported file type')) {
        showNotification(t('unsupportedFileType'));
      } else if (errMsg.includes('No supported Spine') || errMsg.includes('No valid files or models')) {
        showNotification(t('noSupportedModels'));
      } else if (errMsg.includes('Invalid path')) {
        showNotification(t('invalidPath'));
      } else {
        const inputPath = paths[0] || '';
        const isUrl = inputPath.startsWith('http://') || inputPath.startsWith('https://');
        if (isUrl) {
          showNotification(t('invalidUrl'));
        } else {
          showNotification(errMsg);
        }
      }
      showSpinner = false;
    }
  }

  async function initModel(previousSkins = [], options = {}) {
    const detectAlpha = typeof options === 'boolean' ? options : !!options?.detectAlpha;
    currentLoadId++;
    const loadId = currentLoadId;
    const previousAnimation = sidebar?.getSelectedAnimation() ?? '';
    const previousAnimationName = sidebar?.getSelectedAnimationText() || '';
    const { files, selectedDir, selectedScene } = appState.directories;
    if (!files || !selectedDir) return;
    const scenes = files[selectedDir];
    if (!scenes || scenes.length === 0) return;
    const fileNames = scenes[selectedScene];
    const rendererType = getRendererType(fileNames);
    const defaultAlphaMode = rendererType === 'live2d' ? 'pma' : rendererType === 'spine' ? 'npm' : null;
    if (defaultAlphaMode && appState.alphaMode !== defaultAlphaMode) {
      appState.alphaMode = defaultAlphaMode;
      saveSetting('spive2d_alpha_mode', defaultAlphaMode);
    }
    let renderer = await preloadManager.consumePreloaded(selectedDir, fileNames);
    if (renderer) {
      if (typeof renderer.activate === 'function') {
        renderer.activate();
      }
      const oldRenderer = getRenderer();
      if (oldRenderer && oldRenderer !== renderer) {
        oldRenderer.dispose();
        const oldCanvas = oldRenderer.getCanvas();
        if (canvasContainer?.contains(oldCanvas) && oldCanvas !== renderer.getCanvas()) {
          canvasContainer.removeChild(oldCanvas);
        }
      }
    } else {
      const oldRenderer = getRenderer();
      if (oldRenderer) {
        oldRenderer.dispose();
        const oldCanvas = oldRenderer.getCanvas();
        if (canvasContainer?.contains(oldCanvas)) {
          canvasContainer.removeChild(oldCanvas);
        }
        setRenderer(null);
      }
      renderer = createRenderer(fileNames);
      loadingRenderers.push(renderer);
      if (renderer['setAlphaMode']) {
        renderer['setAlphaMode'](appState.alphaMode);
      }
      if (renderer.setTextureFilter) {
        renderer.setTextureFilter(appState.textureFilter);
      }
      try {
        await renderer.load(sceneDir(selectedDir, fileNames), fileNames, { detectAlpha });
      } catch (e) {
        console.error(e);
        loadingRenderers = loadingRenderers.filter(r => r !== renderer);
        return;
      }
      if (loadId !== currentLoadId) {
        loadingRenderers = loadingRenderers.filter(r => r !== renderer);
        renderer.dispose();
        return;
      }
      loadingRenderers = loadingRenderers.filter(r => r !== renderer);
    }
    const canvas = renderer.getCanvas();
    if (canvasContainer && !canvasContainer.contains(canvas)) {
      canvasContainer.appendChild(canvas);
    }
    const detectedAlphaMode = renderer.getAlphaMode?.();
    if (!defaultAlphaMode && detectAlpha && detectedAlphaMode) {
      if (detectedAlphaMode !== appState.alphaMode) {
        appState.alphaMode = detectedAlphaMode;
        saveSetting('spive2d_alpha_mode', detectedAlphaMode);
      }
    } else if (typeof renderer.setAlphaMode === 'function' && renderer.getAlphaMode && renderer.getAlphaMode() !== appState.alphaMode) {
      await renderer.setAlphaMode(appState.alphaMode);
    }
    setRenderer(renderer);
    const rendererCanvas = renderer.getCanvas();
    requestAnimationFrame(() => {
      if (typeof renderer['_revealCanvas'] !== 'function') rendererCanvas.style.opacity = '1';
    });
    const categories = renderer.getPropertyCategories();
    appState.propertyCategory = categories[0] || 'parameters';
    appState.resetTransform();
    appState.resetAnimation();
    if (appState.enableInteractions) {
      sidebar?.setSelectedAnimation('');
      await renderer.setAnimation('');
    }
    if (previousSkins.length > 0 && renderer.getPropertyItems && 'applySkins' in renderer && typeof renderer.applySkins === 'function') {
      const availableSkins = renderer.getPropertyItems('skins') || [];
      const matchingSkins = previousSkins.filter(skinName => availableSkins.some(s => s.name === skinName));
      if (matchingSkins.length > 0) {
        renderer.applySkins(matchingSkins);
      }
    }
    sidebar?.setSelectedExpression('');
    sidebar?.refreshProperties();
    if (appState.enableInteractions) {
      sidebar?.setSelectedAnimation('');
      await renderer.setAnimation('');
      preloadManager.triggerPreload(selectedDir, scenes, selectedScene, { detectAlpha });
      return;
    }
    const animations = renderer.getAnimations();
    const keepSetupPose = appState.initialized && previousAnimation === '';
    if (keepSetupPose) {
      sidebar?.setSelectedAnimation('');
      handleAnimationChange('');
    } else if (animations.length > 0) {
      let targetAnim = animations[0].value;
      let foundMatch = false;
      if (previousAnimationName) {
        const match = animations.find(a => a.name === previousAnimationName);
        if (match) {
          targetAnim = match.value;
          foundMatch = true;
        }
      }
      if (!foundMatch) {
        const idleMatch = findIdleAnimation(animations);
        if (idleMatch) {
          targetAnim = idleMatch.value;
        }
      }
      sidebar?.setSelectedAnimation(targetAnim);
      handleAnimationChange(targetAnim);
    } else {
      sidebar?.setSelectedAnimation('');
      handleAnimationChange('');
    }
    preloadManager.triggerPreload(selectedDir, scenes, selectedScene, { detectAlpha });
  }

  function disposeModel(clearPreload = true) {
    currentLoadId++;
    if (clearPreload) {
      preloadManager.clear();
    }
    const renderer = getRenderer();
    if (renderer) {
      renderer.dispose();
      const canvas = renderer.getCanvas();
      if (canvasContainer?.contains(canvas)) {
        canvasContainer.removeChild(canvas);
      }
      setRenderer(null);
    }
    for (const r of loadingRenderers) {
      r.dispose();
      const canvas = r.getCanvas();
      if (canvasContainer?.contains(canvas)) {
        canvasContainer.removeChild(canvas);
      }
    }
    loadingRenderers = [];
  }

  function handleSceneChange(e) {
    appState.directories.selectedScene = Number(e.target.value);
    const previousSkins = getRenderer()?.getPropertyItems?.('skins')?.filter(item => item.checked).map(item => item.name) || [];
    disposeModel(false);
    initModel(previousSkins);
  }

  function toggleInteractionMode() {
    const renderer = getRenderer();
    if (!appState.enableInteractions) {
      interactionRestoreAnimation = sidebar?.getRestoreAnimation?.() || '';
      appState.enableInteractions = true;
      sidebar?.setSelectedAnimation('');
      renderer?.setAnimation('');
    } else {
      appState.enableInteractions = false;
      const restore = interactionRestoreAnimation || sidebar?.getRestoreAnimation?.() || '';
      sidebar?.setSelectedAnimation(restore);
      renderer?.setAnimation(restore);
    }
  }

  function toggleHitAreaHints() {
    appState.showHitAreas = !appState.showHitAreas;
    saveSetting('spive2d_show_hit_areas', appState.showHitAreas);
  }

  function toggleInteractionCoordinates() {
    appState.showInteractionCoordinates = !appState.showInteractionCoordinates;
    saveSetting('spive2d_show_interaction_coordinates', appState.showInteractionCoordinates);
    if (!appState.showInteractionCoordinates) {
      interactionCoordinate = null;
      coordinateProbePointerId = null;
    }
  }

  function handleInteractionPointerDown(event) {
    if (event.button !== 0) return;
    event.stopPropagation();
    const renderer = getRenderer();
    const target = renderer?.getInteractionTargetAt?.(event.clientX, event.clientY) ?? '';
    if (typeof renderer?.getInteractionTargetAt === 'function' && !target) return;
    interactionStart = {
      x: event.clientX,
      y: event.clientY,
      lastX: event.clientX,
      lastY: event.clientY,
      target,
      pointerId: event.pointerId,
      dragged: false,
    };
    event.currentTarget.setPointerCapture?.(event.pointerId);
  }

  function handleInteractionPointerMove(event) {
    if (!interactionStart || event.pointerId !== interactionStart.pointerId) return;
    event.stopPropagation();
    const dx = event.clientX - interactionStart.lastX;
    const dy = event.clientY - interactionStart.lastY;
    interactionStart.lastX = event.clientX;
    interactionStart.lastY = event.clientY;
    if (!interactionStart.dragged && Math.hypot(event.clientX - interactionStart.x, event.clientY - interactionStart.y) <= 5) return;
    interactionStart.dragged = true;
    getRenderer()?.applyInteractionDrag?.(dx, dy);
  }

  function handleInteractionPointerUp(event) {
    if (!interactionStart || event.button !== 0 || event.pointerId !== interactionStart.pointerId) return;
    event.stopPropagation();
    const { target, dragged, pointerId } = interactionStart;
    interactionStart = null;
    event.currentTarget.releasePointerCapture?.(pointerId);
    if (!dragged) getRenderer()?.triggerInteraction?.(target);
  }

  function handleInteractionPointerCancel(event) {
    if (interactionStart?.pointerId === event.pointerId) interactionStart = null;
  }

  function handleAnimationChange(value) {
    const renderer = getRenderer();
    renderer?.setAnimation(value);
    if (appState.animation.paused) {
      animController?.resetProgress();
      requestAnimationFrame(() => {
        renderer?.seekAnimation(0);
      });
    }
    sidebar?.refreshProperties();
  }

  function handleExpressionChange(value) {
    getRenderer()?.setExpression(value);
  }

  function handleKeyDown(e) {
    if (document.activeElement?.matches('input, textarea')) return;
    if (sceneGridOpen) {
      if (isShortcut(e, shortcuts.toggleSceneGrid)) {
        e.preventDefault();
        sceneGridOpen = false;
      }
      return;
    }
    const key = e.key.toLowerCase();    
    if ((key === 'w' || key === 'q') && (e.ctrlKey || e.metaKey)) {
      e.preventDefault();
      getCurrentWindow().close();
      return;
    }
    if (isShortcut(e, shortcuts.toggleFullscreen)) {
      e.preventDefault();
      toggleFullscreen();
      return;
    }
    if (e.key === 'Escape' && !dialogOpen) {
      exitFullscreen();
      return;
    }
    const extra = Object.keys(EXTRA_SHORTCUTS).find((action) => isShortcut(e, shortcuts[action]));
    if (extra) {
      const { anytime, keepFocus, run } = EXTRA_SHORTCUTS[extra];
      if (!anytime && !appState.initialized) return;
      e.preventDefault();
      if (!keepFocus) focusBody();
      run();
      return;
    }
    if (!isShortcut(e, shortcuts.toggleDialog) && !appState.initialized) return;
    if (isShortcut(e, shortcuts.prevScene)) { navigateScene(-1); }
    else if (isShortcut(e, shortcuts.nextScene)) { navigateScene(1); }
    else if (isShortcut(e, shortcuts.prevAnim)) { sidebar?.navigateAnimation(-1); }
    else if (isShortcut(e, shortcuts.nextAnim)) { sidebar?.navigateAnimation(1); }
    else if (isShortcut(e, shortcuts.togglePause)) { e.preventDefault(); animController?.togglePlayPause(); }
    else if (isShortcut(e, shortcuts.exportImage)) { doExportImage(); }
    else if (isShortcut(e, shortcuts.exportImageSeq)) { doExportImageSequence(); }
    else if (isShortcut(e, shortcuts.exportAnim)) { doExportAnimation(); }
    else if (isShortcut(e, shortcuts.exportModel)) { doExportModel(); }
    else if (isShortcut(e, shortcuts.toggleDialog)) { toggleDialog(); }
    else if (isShortcut(e, shortcuts.toggleSceneGrid)) { e.preventDefault(); openSceneGrid(); }
    else if (isShortcut(e, shortcuts.addToList)) {
      invoke('append_to_list', { text: getSceneText(), customDir: appState.exportDir || null }).then(() => {
        showNotification(t('addedToList'), 'success');
      });
    }
    else { return; }
    focusBody();
  }

  const TEXTURE_FILTERS = ['nearest', 'linear', 'bicubic', 'lanczos'];
  const TEXTURE_FILTER_KEYS = { nearest: 'textureFilterNearest', linear: 'textureFilterLinear', bicubic: 'textureFilterBicubic', lanczos: 'textureFilterLanczos' };

  function notifyInfo(message) {
    showNotification(message, 'info', 1500);
  }

  function toggleSetting(key, storageKey, titleKey) {
    appState[key] = !appState[key];
    saveSetting(storageKey, appState[key]);
    notifyInfo(`${t(titleKey)}: ${appState[key] ? 'ON' : 'OFF'}`);
  }

  function resetTransformValues(values) {
    appState.transform = { ...appState.transform, ...values };
  }

  function resetExportValues(values, titleKey) {
    Object.assign(appState, values);
    notifyInfo(t(titleKey));
  }

  function cycleTextureFilter() {
    const index = TEXTURE_FILTERS.indexOf(appState.textureFilter);
    const next = TEXTURE_FILTERS[(index + 1) % TEXTURE_FILTERS.length];
    appState.textureFilter = next;
    saveSetting('spive2d_texture_filter', next);
    getRenderer()?.setTextureFilter?.(next);
    notifyInfo(`${t('textureFilter')} ${t(TEXTURE_FILTER_KEYS[next])}`);
  }

  const EXTRA_SHORTCUTS = {
    openDirectory: { anytime: true, run: () => settingsDialog?.handleOpenDirectory() },
    openArchive: { anytime: true, run: () => settingsDialog?.handleOpenArchive() },
    openCurrentDirectory: { run: () => settingsDialog?.handleOpenCurrentDir() },
    focusUrl: { anytime: true, keepFocus: true, run: () => settingsDialog?.focusUrl() },
    focusFilter: { keepFocus: true, run: () => sidebar?.focusSceneFilter() },
    prevExpression: { run: () => sidebar?.navigateExpression(-1) },
    nextExpression: { run: () => sidebar?.navigateExpression(1) },
    toggleSetupPose: { run: () => sidebar?.toggleSetupPose() },
    resetOverrides: { run: () => sidebar?.resetOverrides() },
    resetView: { run: () => { appState.resetTransform(); getRenderer()?.resetTransform(); } },
    resetScale: { run: () => resetTransformValues({ scale: 1 }) },
    resetRotation: { run: () => resetTransformValues({ rotate: 0 }) },
    resetPositionX: { run: () => resetTransformValues({ moveX: 0 }) },
    resetPositionY: { run: () => resetTransformValues({ moveY: 0 }) },
    resetPosition: { run: () => resetTransformValues({ moveX: 0, moveY: 0 }) },
    cycleTextureFilter: { run: cycleTextureFilter },
    openBackgroundImage: { anytime: true, run: () => settingsDialog?.handleOpenImage() },
    removeBackground: { anytime: true, run: () => settingsDialog?.handleRemoveBackground() },
    openExportDirectory: { anytime: true, run: () => openExportDirectory() },
    resetExportMarginX: { anytime: true, run: () => resetExportValues({ exportMarginX: 0 }, 'shortcutResetExportMarginX') },
    resetExportMarginY: { anytime: true, run: () => resetExportValues({ exportMarginY: 0 }, 'shortcutResetExportMarginY') },
    resetExportMargin: { anytime: true, run: () => resetExportValues({ exportMarginX: 0, exportMarginY: 0 }, 'shortcutResetExportMargin') },
    resetExportScale: { anytime: true, run: () => resetExportValues({ exportScale: 100 }, 'shortcutResetExportScale') },
    toggleIdle: { anytime: true, run: () => toggleSetting('enableIdle', 'spive2d_enable_idle', 'shortcutToggleLive2DIdle') },
    toggleBreathing: { anytime: true, run: () => toggleSetting('enableBreathing', 'spive2d_enable_breathing', 'shortcutToggleLive2DBreathing') },
    toggleBlinking: { anytime: true, run: () => toggleSetting('enableBlinking', 'spive2d_enable_blinking', 'shortcutToggleLive2DBlinking') },
    toggleHitAreas: { anytime: true, run: () => toggleSetting('showHitAreas', 'spive2d_show_hit_areas', 'shortcutToggleLive2DHitAreas') },
    toggleMouseTracking: { anytime: true, run: () => toggleSetting('enableMouseTracking', 'spive2d_enable_mouse_tracking', 'shortcutToggleLive2DMouseTracking') },
    togglePhysics: { anytime: true, run: () => toggleSetting('enablePhysics', 'spive2d_enable_physics', 'shortcutToggleLive2DPhysics') },
  };

  function isShortcut(e, key) {
    if (!key) return false;
    return key.length === 1 ? e.key.toLowerCase() === key.toLowerCase() : e.key === key;
  }

  function navigateScene(delta) {
    const { files, selectedDir, selectedScene, sceneFilter } = appState.directories;
    const indices = filterSceneIndices(files[selectedDir] || [], sceneFilter);
    const pos = indices.indexOf(selectedScene);
    if (indices.length === 0 || (pos !== -1 && indices.length === 1)) return;
    let newPos;
    if (pos !== -1) {
      newPos = (pos + delta + indices.length) % indices.length;
    } else if (delta > 0) {
      newPos = Math.max(0, indices.findIndex(i => i > selectedScene));
    } else {
      newPos = indices.findLastIndex(i => i < selectedScene);
      if (newPos === -1) newPos = indices.length - 1;
    }
    handleSceneChange({ target: { value: indices[newPos] }});
  }

  function toggleDialog() {
    if (showSpinner) return;
    dialogOpen = !dialogOpen;
  }

  function openSceneGrid() {
    if (showSpinner || dialogOpen) return;
    const { files, selectedDir } = appState.directories;
    if (!files?.[selectedDir]?.length) return;
    sceneGridOpen = true;
  }

  function focusBody() {
    if (document.activeElement !== document.body) {
      if (document.activeElement instanceof HTMLElement) {
        document.activeElement.blur();
      }
      document.body.focus();
    }
  }

  function doExportImage() {
    const sceneText = getSceneText();
    let animText = sidebar?.getSelectedAnimationText() || '';
    if (!animText && sidebar) {
      animText = sidebar.getSelectedExpressionText() || '';
    }
    exportImage(sceneText, animText);
  }

  function doExportAnimation() {
    if (getRenderer()?.rendererType === 'layered') {
      return;
    }
    const sceneText = getSceneText();
    const animText = sidebar?.getSelectedAnimationText() || '';
    const animValue = sidebar?.getSelectedAnimation?.() || '';
    const exprValue = sidebar?.getSelectedExpression?.() || '';
    exportAnimation(sceneText, animText, animValue, exprValue);
  }

  function doExportModel() {
    exportModelFiles(getSceneText());
  }

  async function doExportImageSequence() {
    if (getRenderer()?.rendererType === 'layered') {
      return;
    }
    const sceneText = getSceneText();
    const animText = sidebar?.getSelectedAnimationText() || '';
    const animValue = sidebar?.getSelectedAnimation?.() || '';
    const safeName = animText ? animText.split('.')[0] : 'sequence';
    const exportBaseDir = await getExportDirectory();
    const folderName = `${sanitizeFilename(sceneText)}_${sanitizeFilename(safeName)}`;
    const targetDir = await join(exportBaseDir, folderName);
    try {
      await mkdir(targetDir, { recursive: true });
    } catch (err) {
      console.error('Failed to create export directory:', err);
    }
    const exprValue = sidebar?.getSelectedExpression?.() || '';
    exportImageSequence(targetDir, sceneText, animText, animValue, exprValue);
  }

  function getSceneText() {
    const scenes = appState.directories.files?.[appState.directories.selectedDir] || [];
    const currentSceneStr = scenes.length > 0 && appState.directories.selectedScene >= 0 && appState.directories.selectedScene < scenes.length ? scenes[appState.directories.selectedScene].name : '';
    return currentSceneStr ? currentSceneStr.split('/').filter(Boolean).pop().replace(/^\u200B/, '') : 'scene';
  }

  function handleResize() {
    const w = window.innerWidth;
    const h = window.innerHeight;
    appState.viewport = { width: w, height: h };
    const renderer = getRenderer();
    if (renderer && appState.initialized) {
      renderer.resize(w, h);
      renderer.applyTransform(
        appState.transform.scale,
        appState.transform.moveX,
        appState.transform.moveY,
        appState.transform.rotate
      );
    }
  }

  function handleContextMenu(e) {
    e.preventDefault();
  }
</script>

<svelte:window
  onresize={handleResize}
  onkeydown={handleKeyDown}
/>

{#if showSpinner}
  <div id="spinner-backdrop">
    <div id="spinner"></div>
  </div>
{/if}

<div use:transformAction={{ appState, sidebar, animController, dialogOpen: dialogOpen || sceneGridOpen }}>
  <SettingsDialog bind:this={settingsDialog} bind:open={dialogOpen} onPathSelected={processPath} onShortcutsChanged={refreshShortcuts} />
  <Sidebar
    bind:this={sidebar}
    onSceneChange={handleSceneChange}
    onAnimationChange={handleAnimationChange}
    onExpressionChange={handleExpressionChange}
    onSettingsClick={() => dialogOpen = true}
    onSceneGridClick={() => sceneGridOpen = true}
  />
  <SceneThumbnailGrid bind:open={sceneGridOpen} onSelect={(index) => handleSceneChange({ target: { value: index } })} />
  <div id="canvasContainer" bind:this={canvasContainer}></div>
  {#if appState.enableInteractions}
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <div
      id="interactionLayer"
      onpointerdown={handleInteractionPointerDown}
      onpointermove={handleInteractionPointerMove}
      onpointerup={handleInteractionPointerUp}
      onpointercancel={handleInteractionPointerCancel}
    ></div>
  {/if}
  {#if appState.initialized}
    <button
      id="interactionModeButton"
      class:active={appState.enableInteractions}
      onclick={toggleInteractionMode}
      title={appState.enableInteractions ? t('disableInteractionMode') : t('enableInteractionMode')}
      aria-pressed={appState.enableInteractions}
    >
      <svg viewBox="0 0 24 24" width="18" height="18" fill="currentColor" aria-hidden="true">
        <path d="M9 11.24V7.5a2.5 2.5 0 0 1 5 0v3.15l1.1-.44a2.02 2.02 0 0 1 2.64 1.17l.66 1.82c1.1 3.03-.49 6.38-3.52 7.48l-.28.1c-1.58.58-3.34.58-4.92 0l-.28-.1c-3.03-1.1-4.62-4.45-3.52-7.48l.66-1.82a2.02 2.02 0 0 1 2.64-1.17l-.18.03zm2-3.74V11h1V7.5a.5.5 0 0 0-1 0z"/>
      </svg>
    </button>
    <button
      id="hitAreaHintButton"
      class:active={appState.showHitAreas}
      onclick={toggleHitAreaHints}
      title={t(appState.showHitAreas ? 'hideHitAreas' : 'showHitAreas')}
      aria-label={t(appState.showHitAreas ? 'hideHitAreas' : 'showHitAreas')}
      aria-pressed={appState.showHitAreas}
    >
      <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" aria-hidden="true">
        <path d="M5 5h5v5H5zM14 4l6 3.5v7L14 18l-6-3.5v-7L14 4zM5 15h5v5H5z"/>
      </svg>
    </button>
    <button
      id="coordinateProbeButton"
      class:active={appState.showInteractionCoordinates}
      onclick={toggleInteractionCoordinates}
      title={t(appState.showInteractionCoordinates ? 'hideInteractionCoordinates' : 'showInteractionCoordinates')}
      aria-label={t(appState.showInteractionCoordinates ? 'hideInteractionCoordinates' : 'showInteractionCoordinates')}
      aria-pressed={appState.showInteractionCoordinates}
    >
      <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" aria-hidden="true">
        <path d="M3 12h17m-4-4 4 4-4 4M12 21V4m-4 4 4-4 4 4"/>
      </svg>
    </button>
  {/if}
  <HitAreaOverlay />
  <InteractionCoordinateOverlay coordinate={interactionCoordinate} />
  <PickOverlay dialogOpen={dialogOpen || sceneGridOpen} onChange={() => sidebar?.refreshProperties()} />
</div>

<AnimationController bind:this={animController} />

<ExportQueue />
<Notification />

<style>
  #spinner-backdrop {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.4);
    backdrop-filter: blur(3px);
    z-index: var(--z-spinner);
  }

  #spinner {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 80px;
    height: 80px;
    border: 8px solid #eee;
    border-top: 8px solid #888;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }

  #canvasContainer {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: var(--z-model);
  }

  #canvasContainer :global(canvas) {
    position: absolute;
    top: 0;
    left: 0;
    z-index: 1;
  }

  #interactionLayer {
    position: fixed;
    top: 0;
    left: var(--sidebar-width);
    right: var(--sidebar-width);
    bottom: 80px;
    z-index: var(--z-interaction);
    background: transparent;
    cursor: pointer;
    touch-action: none;
  }

  #interactionModeButton {
    position: fixed;
    top: 12px;
    right: 12px;
    z-index: var(--z-control);
    width: 38px;
    height: 38px;
    border: var(--border-color);
    border-radius: 50%;
    background: var(--sidebar-color);
    color: #ccc;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.25);
  }

  #interactionModeButton:hover,
  #interactionModeButton.active,
  #hitAreaHintButton:hover,
  #hitAreaHintButton.active,
  #coordinateProbeButton:hover,
  #coordinateProbeButton.active {
    background: #555;
    color: #fff;
  }

  #hitAreaHintButton,
  #coordinateProbeButton {
    position: fixed;
    right: 12px;
    z-index: var(--z-control);
    width: 38px;
    height: 38px;
    padding: 0;
    border: var(--border-color);
    border-radius: 50%;
    background: var(--sidebar-color);
    color: #ccc;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.25);
  }

  #hitAreaHintButton {
    bottom: 92px;
  }

  #coordinateProbeButton {
    bottom: 138px;
  }

  #hitAreaHintButton:hover,
  #coordinateProbeButton:hover {
    color: #fff;
  }

  #coordinateProbeButton:focus-visible,
  #hitAreaHintButton:focus-visible,
  #interactionModeButton:focus-visible {
    outline: 2px solid #ffc440;
    outline-offset: 2px;
  }
</style>
