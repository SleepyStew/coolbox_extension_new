<template>
    <!-- Edit Mode -->
    <div class="grid-layout" @click="clearSelectedComponent" v-if="editMode">
        <Container v-for="[column, components] in Object.entries(currentPageLayout)"
                   group-name="homepage"
                   :data-col="column"
                   @drop="(ev) => dropComponent(column as Column, ev)"
                   :get-child-payload="(ev) => getPayload(column as Column, ev)"
                   non-drag-area-selector=".no-drag"
                   :drop-placeholder="{className: 'irrelevant'}"
        >
            <Draggable v-for="[i, el] in Object.entries(components)"
                       @click="selectComponent"
                       :data-i="i" :key="'d' + i">
                <component :is="el" :key="i" :widg-info="{edit: true, col: column, add: false}" @delete="deleteSelectedWidget"/>
            </Draggable>
        </Container>
    </div>
    <!-- Normal Mode -->
    <div class="grid-layout relative" v-else>
        <div class="absolute right-0 -top-2">
            <div class="dui-indicator">
                <div class="dui-tooltip bg-transparent z-[1003] before:-translate-x-32" data-tip="Customise Homepage">
                    <button class="cb-icon-button cb-icon text-themeText bg-primary rounded-full" @click="enterEditMode">edit</button>
                </div>
            </div>
        </div>
        <div v-for="[column, components] in Object.entries(currentPageLayout)">
            <component v-for="[i, el] in Object.entries(components)"
                       :is="el" :key="i" :class="{'px-2': true, 'slide-in': pageHasBeenEdited}"
                       :widg-info="{edit: false, col: column, add: false, reminders: reminders}"
                       class="my-2"
                       @popup="openPopup"
            />
        </div>
    </div>

    <teleport to="body">
        <ReminderPopup :edit="false" ref="createReminderPopup"/>
        <ReminderPopup :edit="true" ref="editReminderPopup"/>
        <ViewRemindersPopup ref="viewReminderPopup" @edit-reminder="(rem) => {editReminderPopup.openPopup(rem)}"/>
        <TaskPopup ref="taskPopup" :subjects="subjects"/>
    </teleport>

    <!-- Page Editing Toast -->
    <ThemePicker v-if="editMode"/>
    <Shadow>
        <div class="dui-toast z-[1005]" v-if="editMode">
            <div class="dui-alert p-2 shadow-2xl shadow-black">
                <span class="cb-icon text-lg">edit</span>
                <span>You are in edit mode! Click a widget to<br>select it and edit it, or drag them around.</span>
                <div class="dui-tooltip" data-tip="Reset Layout">
                    <button class="dui-btn bg-gray-300" @click="resetPageLayout">
                        <span class="cb-icon text-lg">restart_alt</span>
                    </button>
                </div>
                <div class="dui-indicator">
                    <div class="dui-tooltip" data-tip="Add Widgets">
                        <button class="dui-btn dui-btn-secondary" @click="drawerOpen = !drawerOpen">
                            <span class="cb-icon text-lg">add</span>
                        </button>
                    </div>
                </div>
                <div class="dui-tooltip" data-tip="Done">
                    <button class="dui-btn dui-btn-primary" @click="editMode = false; clearSelectedComponent();">
                        <span class="cb-icon text-lg">done</span>
                    </button>
                </div>
            </div>
        </div>
    </Shadow>

    <!-- Generic Success Toast -->
    <div class="dui-toast -right-1/4 z-[1003]" id="toast-success">
        <div class="border-green-500 notif">
            <span class="cb-icon text-green-500">check</span>
            <span class="content"></span>
        </div>
    </div>

    <!-- Generic Failure Toast -->
    <div class="dui-toast -right-1/4 z-[1003]" id="toast-failure">
        <div class="border-red-400 notif">
            <span class="cb-icon text-red-400">close</span>
            <span class="content"></span>
        </div>
    </div>

    <!-- Widget Add Sidebar -->
    <div :class="{'widget-sidebar': true, 'drawerOpen': drawerOpen}" v-if="editMode">
        <h1>Add Widgets</h1>
        <Container group-name="homepage" behaviour="copy"
                   @drag-start="drawerOpen = false"
                   :get-child-payload="(ev) => allWidgets[ev]"
                   :get-ghost-parent="getBody"
        >
            <Draggable v-for="[i, el] in Object.entries(allWidgets)" :key="'d' + i">
                <component :is="el" :key="i" :widg-info="{edit: true, col: null, add: true}" @click.prevent/>
            </Draggable>
        </Container>
    </div>
</template>

<script setup lang="ts">
import GreetingText from "~/components/widgets/GreetingText.vue";
import UpcomingDueWork from "~/components/widgets/DueWork/UpcomingDueWork.vue";
import Tiles from "~/components/widgets/Tiles.vue";
import CoolBoxMessage from "~/components/widgets/CoolBoxMessage.vue";
import Timetable from "~/components/widgets/Timetable/Timetable.vue";
import TimeLeft from "~/components/widgets/TimeLeft.vue";
import NewsItems from "~/components/widgets/News.vue";
import TermDates from "~/components/widgets/TermDates.vue";
import WeatherWidget from "~/components/widgets/WeatherWidget.vue";
import AnalogClock from "~/components/widgets/Clock.vue";
import Calendar from "~/components/widgets/Calendar.vue";
import DailyVerse from "~/components/widgets/DailyVerse.vue";

import ReminderPopup from "~/components/popups/ReminderPopup.vue";
import ViewRemindersPopup from "~/components/popups/ViewRemindersPopup.vue";
import ThemePicker from "~/components/other/ThemePicker.vue";
import Shadow from "~/components/other/Shadow.vue";

import '@vuepic/vue-datepicker/dist/main.css';

import browser from "webextension-polyfill";
import {reminders} from "~/utils/apiUtils";
import {markRaw, Ref, ref} from "vue";
import type {Component, Raw} from "vue";
import {manualStorageSet} from "~/utils/componentUtils";
import {Reminder, Task} from "~/utils/types";
// types do not exist for this package
// @ts-ignore
import {Container, Draggable} from "vue3-smooth-dnd";
import TodoListWidget from "~/components/widgets/TodoListWidget.vue";
import TaskPopup from "~/components/popups/TaskPopup.vue";

const currentPageLayout: Ref<Record<Column,Component[]>> = ref({
    leftCol: [],
    rightCol: []
})

defineProps<{
    subjects: {name: string, pretty: string}[];
}>();

const createReminderPopup = ref(null);
const editReminderPopup = ref(null);
const viewReminderPopup = ref(null);
const taskPopup = ref(null);

function openPopup(popupType: "createReminder" | "editReminder", payload: Reminder): void;
function openPopup(popupType: "createTask" | "editTask", payload: Task): void;
function openPopup(popupType: "viewReminders"): void;
function openPopup(popupType: string, payload?: Reminder | Task): void {
    switch (popupType) {
        case "viewReminders":
            return viewReminderPopup.value.openPopup();
        case "createReminder":
            return createReminderPopup.value.openPopup(payload as Reminder);
        case "editReminder":
            return editReminderPopup.value.openPopup(payload as Reminder);
        case "createTask":
            return taskPopup.value.openPopup(payload as Task);
        case "editTask":
            return taskPopup.value.openPopup(payload as Task);
        default:
            throw new Error(`Unknown popup type: ${popupType}`);
    }
}

function getBody() {
    return document.body;
}

const widgetMappings = {}

function componentNameToComponent(name: string): Component {
    return allWidgets.find(
        widget => {
            if (name in widgetMappings) {
                return widget['__name'] === widgetMappings[name];
            }
            return widget['__name'] === name;
        }
    )
}

// Get the homepage layout from storage, and if it exists, restore it (or default)
browser.storage.local.get("homepageLayout").then(layout => {
    if (layout.homepageLayout) {
        currentPageLayout.value = {
            leftCol: layout.homepageLayout.leftCol.map(componentNameToComponent),
            rightCol: layout.homepageLayout.rightCol.map(componentNameToComponent)
        }
    } else {
        resetPageLayout();
    }
})

function resetPageLayout() {
    currentPageLayout.value = {
        leftCol: [
            GreetingText,
            Timetable,
            TimeLeft,
            Tiles,
            CoolBoxMessage,
        ].map(markRaw),
        rightCol: [
            UpcomingDueWork,
            NewsItems,
        ].map(markRaw)
    }
    saveLayout();
}

const allWidgets: Raw<Component>[] = [
    DailyVerse,
    AnalogClock,
    WeatherWidget,
    GreetingText,
    Timetable,
    TimeLeft,
    Tiles,
    CoolBoxMessage,
    UpcomingDueWork,
    NewsItems,
    Calendar,
    TermDates,
    TodoListWidget
].map(markRaw);
const drawerOpen = ref(false);

function saveLayout() {
    const layout = {
        leftCol: currentPageLayout.value.leftCol.map(el => el['__name']),
        rightCol: currentPageLayout.value.rightCol.map(el => el['__name']),
    }
    manualStorageSet({
        "homepageLayout": layout
    });
}

type DropInfo = {
    removedIndex: number | null,
    addedIndex: number | null,
    payload: Raw<Component>
}

type Column = "leftCol" | "rightCol";

function dropComponent(column: Column, dropInfo: DropInfo) {
    // If there was an element removed, delete it from the page.
    if (dropInfo.removedIndex !== null) {
        currentPageLayout.value[column].splice(dropInfo.removedIndex, 1);
    }
    // If there was an element added, add it to the page.
    if (dropInfo["addedIndex"] !== null) {
        currentPageLayout.value[column].splice(dropInfo.addedIndex, 0, dropInfo.payload);
    }
    saveLayout();
}

function deleteSelectedWidget() {
    const column: Column = selectedElement.parentElement.dataset.col;
    const i = Number(selectedElement.dataset.i);

    currentPageLayout.value[column].splice(i, 1);
    clearSelectedComponent();
    saveLayout();
}

function getPayload(col: Column, index: number) {
    return currentPageLayout.value[col][index];
}

let selectedElement;
const editMode = ref(false);
const pageHasBeenEdited = ref(false);

function enterEditMode() {
    editMode.value = true;
    pageHasBeenEdited.value = true;
}

function selectComponent(event) {
    if (editMode.value) {
        if (!event.target.matches(".widget-settings *")) {
            event.preventDefault();
            event.stopPropagation();

            const component = event.currentTarget;

            selectedElement?.classList.remove("selected");
            selectedElement = component;

            component.classList.add("selected");
        } else {
            event.stopPropagation();
        }
    }
}

function clearSelectedComponent() {
    selectedElement?.classList.remove("selected");
    selectedElement = null;
    drawerOpen.value = false;
}
</script>

<!--suppress CssUnusedSymbol -->
<style>
.cb-icon-button {
    @apply bg-transparent border-0 text-2xl text-gray-800 hover:bg-gray-200 hover:text-gray-600 aspect-square rounded-md leading-6 m-0 z-[1];
}
.selected, .smooth-dnd-drop-preview-constant-class {
    @apply outline-blue-500 outline-2 outline outline-offset-2 rounded-sm;
}
.smooth-dnd-drop-preview-constant-class {
    margin-top: 4px;
    border-radius: 8px;
    outline-offset: -3px;
}
.grid-layout {
    @apply grid grid-cols-[65%,35%] gap-2 relative;
}

.smooth-dnd-draggable-wrapper {
    @apply bg-white drop-shadow rounded-md mb-2 p-2 cursor-move animate-[slide-down_400ms_ease-out];
}

.slide-in {
    @apply animate-[slide-up_300ms_ease-out];
}

.mt {
    @apply mt-3
}

@keyframes slide-down {
    from {
        transform: translateY(-20%);
    }
    to {
        transform: translateY(0%);
    }
}

@keyframes slide-up {
    from {
        transform: translateY(20%);
    }
    to {
        transform: translateY(0%);
    }
}

@keyframes slide-x {
    from {
        transform: translateX(-100%);
    }
    to {
        transform: translateX(0);
    }
}

.widget-sidebar {
    height: 100vh;
    width: 40%;
    overflow: clip scroll;
    @apply fixed bg-primary left-0 top-0 shadow-2xl z-[1002] p-6 -translate-x-full;
}
.drawerOpen {
    animation: slide-x 500ms forwards;
}
.drawerClosed {
    animation: slide-x 500ms reverse;
}
.dui-toggle {
    @apply !dui-toggle !static !opacity-100 border-solid border-gray-500 !m-0
}
.right-off-canvas-menu {
    z-index: 1004;
}
.dp__input_icon_pad {
    padding-left: var(--dp-input-icon-padding) !important;
}
.notif {
    @apply dui-alert p-2 shadow-2xl shadow-black bg-primary text-themeText border-solid;
}
</style>