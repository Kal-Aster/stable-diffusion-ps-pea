<script lang="ts">

import type { Slots } from "@vue/runtime-core";

interface ImageItem {
    imageURL: string;
    name: string;
};

export default {
    name: 'ImagePicker',
    props: {
        images: {
            type: Array<ImageItem>,
            default: [],
        },
        selectedImages: {
            type: Array<string>,
            default: [],
        },
        displayNames: {
            type: Boolean,
            default: true,
        },
        actionsOnHover: {
            type: Boolean,
            default: false,
        },
    },
    emits: ['item-clicked'],
    setup(props, { emit }) {
        return {
            onItemClicked(item: ImageItem) {
                emit('item-clicked', item);
            },
            hasActionsSlot(slots: Slots) {
                return "default" in slots
            }
        };
    },
};
</script>

<template>
    <div class="image-grid">
        <div :class="{ 'grid-item': true }"
            v-for="(item, index) in images" :key="index" @click="onItemClicked(item)">
            <div :class="{ 'image-preview': true, 'selected': selectedImages!.includes(item.name) }"
                :style="`background-image: url(${item.imageURL});`"></div>
            <div
                :class="{
                    'actions': true,
                    'display-on-hover': actionsOnHover
                }"
                :hidden="!displayNames && !hasActionsSlot($slots)">
                <span class="name" v-if="displayNames">{{ item.name }}</span>
                <slot :image="item"></slot>
            </div>
        </div>
    </div>
</template>

<style scoped>
.selected {
    border: 4px solid #88a950 !important;
}

.image-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
    grid-gap: 8px;
    overflow: unset;
}

.grid-item {
    overflow: hidden;
    background-size: cover;
    width: 100%;
    height: 0;
    padding-bottom: 100%;
    position: relative;

    border: 1px solid rgb(45, 45, 45);
    border-radius: 3px;
    outline: none;
    transition: box-shadow 200ms cubic-bezier(0.215, 0.61, 0.355, 1) 0s, scale 400ms cubic-bezier(0.215, 0.61, 0.355, 1) 0s;
    margin: 0px;
}

.grid-item:hover {
    border-color: rgb(238, 238, 238);
    border-width: 1px;
    box-shadow: rgb(238, 238, 238) 0px 0px 0px 1px;
}

.grid-item:not(:hover) .display-on-hover {
    display: none;
}

.grid-item .image-preview {
    background: url(image_alt.png) no-repeat center/contain;
    /* Firefox */
    image-rendering: crisp-edges;
    /* Chromium + Safari */
    image-rendering: pixelated;
    inset: 0;
    position: absolute;
}

.actions {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    padding: 0.5em;
    background: linear-gradient(180deg, rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.8));
    box-shadow: 0 0 0.25em 0.25em rgba(0, 0, 0, 0.5);
    text-shadow: 0 0 0.2em black;
}

.actions * {
    color: white;
}

.actions .name {
    font-weight: bold;
    line-break: anywhere;
    overflow: hidden;
    display: block;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.grid-item:hover .actions .name {
    overflow: visible;
    text-overflow: unset;
    white-space: normal;
}
</style>


