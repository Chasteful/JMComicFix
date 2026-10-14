<script lang="ts">
    import ClickGui from "./ClickGui.svelte";
    import GlobalSettings from "./tabs/GlobalSettings.svelte";
    import Tabs from "./tabs/Tabs.svelte";
    import {gridSize, os, scaleFactor, showGrid, snappingEnabled, panelLength,fontSize,darken} from "./clickgui_store";
    import type {ConfigurableSetting, TogglableSetting} from "../../integration/types";
    import {onDestroy, onMount} from "svelte";
    import {
        getClientInfo,
        getGameWindow,
        getModuleSettings,
        setHudEditorSelected,
        setTyping
    } from "../../integration/rest";
    import {listen} from "../../integration/ws";
    import type {ClickGuiValueChangeEvent} from "../../integration/events";
    import {resolutionCoefficient} from "../../util/resolution_utils";
    import HudEditor from "./tabs/hud_editor/HudEditor.svelte";
    import {isTextEntry} from "../../util/utils";
    import {persistentDataLoaded, setItem} from "../../integration/persistent_storage";

    const TAB_KEY = "clickgui.tab";

    const tabs = [
        {title: "ClickGUI", content: ClickGui},
        {title: "HUD Editor", content: HudEditor},
        {title: "Settings", content: GlobalSettings},
    ];

    let activeTab = $state(0);
    let tabRestored = false;
    let minecraftScaleFactor = $state(2);
    let clickGuiScaleFactor = $state(1);
    let panelLengthFactor = $state(64)
    let fontSizeFactor = $state(14)

    $effect(() => {
        updateScaleFactor()
        $panelLength = panelLengthFactor
        $fontSize = fontSizeFactor

    });

    function updateScaleFactor() {
        $scaleFactor =
            minecraftScaleFactor *
            clickGuiScaleFactor *
            $resolutionCoefficient;
    }

    $effect(() => {
        const title = tabs[activeTab].title;
        if (tabRestored) {
            setItem(TAB_KEY, title);
        }
    });

    async function restoreTab() {
        await persistentDataLoaded;
        const stored = tabs.findIndex(tab => tab.title === localStorage.getItem(TAB_KEY));
        if (stored >= 0) {
            activeTab = stored;
        }
        tabRestored = true;
    }

    /**
     * The client has to know whether the user is typing, otherwise key presses
     * that are meant for a text field are also handled as game input - see
     * `ModuleClickGui.isInSearchBar`.
     *
     * Focus events bubble, so tracking them here covers every text field of the
     * ClickGUI, including the ones rendered by the HUD editor.
     */
    async function handleFocusIn(event: FocusEvent) {
        await setTyping(isTextEntry(event.target));
    }

    async function handleFocusOut(event: FocusEvent) {
        // `relatedTarget` is the element that receives the focus, so moving
        // from one text field to another does not report a pause in typing.
        await setTyping(isTextEntry(event.relatedTarget));
    }

    function applyValues(configurable: ConfigurableSetting) {
        const scaleValue = configurable.value.find(v => v.name === "Scale");
        const panelLength= configurable.value.find(v => v.name === "Length")
        const fontSize = configurable.value.find(v => v.name === "FontSize")
        const snappingValue = configurable.value.find(v => v.name === "Snapping") as TogglableSetting | undefined;

        if (scaleValue) {
            clickGuiScaleFactor = scaleValue.value as number;
        }

        if (panelLength) {
            panelLengthFactor = panelLength.value as number;
        }
        if (fontSize) {
            fontSizeFactor = fontSize.value as number
        }
        if (snappingValue) {
            $snappingEnabled = snappingValue.value.find(v => v.name === "Enabled")?.value as boolean ?? true;
            $gridSize = snappingValue.value.find(v => v.name === "GridSize")?.value as number ?? 10;
        }
    }

    fontSize.subscribe((fontSize) => {
        if (typeof document === 'undefined') return;
        document.documentElement.style.setProperty('--font-size', `${fontSize}px`);
    });

    const handleResize = () => {
        requestAnimationFrame(() => {
            updateScaleFactor()
        });
    };

    onMount(async () => {
        updateScaleFactor()
        await setHudEditorSelected(false);
        restoreTab();

        $os = (await getClientInfo()).os;

        const clickGuiSettings = await getModuleSettings("ClickGUI");
        applyValues(clickGuiSettings);

        await setTyping(false);

        window.addEventListener("resize", handleResize);

    });

    onDestroy(() => {
        window.removeEventListener("resize", handleResize);
    });

    listen("clickGuiValueChange", (e: ClickGuiValueChangeEvent) => {
        applyValues(e.configurable);
    });
</script>

<svelte:window onfocusin={handleFocusIn} onfocusout={handleFocusOut}/>

<div
        class="tabbed-clickgui"
        class:darken={$darken}
>
    <Tabs {tabs} bind:activeTab/>
</div>

<style lang="scss">
  .tabbed-clickgui {
    overflow: hidden;
    position: absolute;
    inset: 0;
    transition: ease background-color .2s;

    &.darken {
      background-color: var(--clickgui-overlay-background-color);
    }
  }
</style>
