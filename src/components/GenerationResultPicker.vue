<script lang="ts">
import { photopeaContext, type PhotopeaBound, boundWidth, boundHeight } from '@/Photopea';
import {
    computed, reactive,
    ref, watch,
    onMounted, onBeforeUpdate,
} from 'vue';
import ImagePicker from './ImagePicker.vue';
import DiceOutlined from './svg/DiceOutlined.vue';
import { CloseOutlined, CheckOutlined, RedoOutlined, DeleteOutlined, ArrowLeftOutlined } from '@ant-design/icons-vue';
import { getImageDimensions, resizeImage } from '@/ImageUtil';
import { type IGeneratedImage } from '@/Automatic1111';
import { ImageResultDestination } from '@/Core';

interface ImageItem extends IGeneratedImage {
    imageURL: string;
    name: string;
};

export default {
    name: 'GenerationResultPicker',
    props: {
        images: {
            type: Array<IGeneratedImage>,
            required: true,
        },
        bound: {
            type: Array<number>,
            required: false,
        },
        maskBlur: {
            type: Number,
            required: false,
        },
        resultDestination: {
            type: Number, // ImageResultDestination
            required: true,
        },
        opened: {
            type: Boolean,
            default: false,
        },
    },
    components: {
        ImagePicker,
        CloseOutlined,
        CheckOutlined,
        RedoOutlined,
        DiceOutlined,
    },
    emits: [
        'result-finalized',
        'generate-more',
        'generate-more-variants',
        'update:opened'
    ],
    setup(props, { emit }) {
        const resultImageItems = computed(() => {
            return props.images.map((image, index) => {
                return {
                    imageURL: image.url,
                    name: `result-${index}`,
                    ...image,
                };
            });
        });
        const selectedResultImages: ImageItem[] = reactive([]);
        const selectedResultImageNames = computed(() => {
            return selectedResultImages.map(image => image.name);
        });

        const photopeaInProgress = ref<boolean>(false);
        async function switchResultImage(imageItem: ImageItem) {
            if (photopeaInProgress.value) return;

            let imageIndex = selectedResultImages.indexOf(imageItem);
            if (imageIndex >= 0) {
                if (!ctrlPressed.value) {
                    return;
                }
                selectedResultImages.splice(imageIndex, 1);
                photopeaInProgress.value = true;
                await photopeaContext.executeTask(async () => {
                    await deselectResultImage();
                    let selResLength = selectedResultImages.length
                    if (selResLength == 0) {
                        return;
                    }
                    await selectResultImage(selectedResultImages[selResLength - 1]);
                });
                photopeaInProgress.value = false;
                return;
            }
            photopeaInProgress.value = true;
            await photopeaContext.executeTask(async () => {
                await deselectResultImage();
                await selectResultImage(imageItem);
            });
            photopeaInProgress.value = false;

            if (!ctrlPressed.value) {
                selectedResultImages.length = 0;
            }
            selectedResultImages.push(imageItem);
        }

        // Thead unsafe. Need to be called within task.
        async function deselectResultImage() {
            // Remove ResultTempLayer (Deselect previous item).
            await photopeaContext.invoke('removeTopLevelLayer', 'ResultTempLayer');
        }
        // Thead unsafe. Need to be called within task.
        async function selectResultImage(imageItem: ImageItem, layerName: string = 'ResultTempLayer') {
            async function newCanvasBound(): Promise<PhotopeaBound> {
                const { width, height } = await getImageDimensions(imageItem.imageURL);
                return [0, 0, width, height] as PhotopeaBound;
            }

            const bound = props.resultDestination === ImageResultDestination.kCurrentCanvas ?
                props.bound! as PhotopeaBound :
                await newCanvasBound();

            await photopeaContext.pasteImageOnPhotopea(
                await resizeImage(imageItem.imageURL, boundWidth(bound), boundHeight(bound)),
                bound, layerName
            );
        }
        function finalizeSelection() {
            selectedResultImages.length = 0;
            emit('result-finalized');
        }
        async function pickSelectedResultImages() {
            if (photopeaInProgress.value) return;
            photopeaInProgress.value = true;
            await photopeaContext.executeTask(async () => {
                if (props.resultDestination === ImageResultDestination.kCurrentCanvas) {
                    await deselectResultImage();
                    if (props.maskBlur) {
                        await photopeaContext.invoke('applyMaskBlur', props.maskBlur);
                    }
                    for (const image of selectedResultImages) {
                        await selectResultImage(image, /* layerName= */'ResultLayer');
                        await photopeaContext.invoke('cropSelectedRegion');
                    }
                    await photopeaContext.invoke('deselect');
                } else { // kNewCanvas
                    // No selection on new canvas, so avoid any operations involving
                    // selection.
                    await deselectResultImage();
                    for (const image of selectedResultImages) {
                        await selectResultImage(image, /* layerName= */'ResultLayer');
                    }
                }
            });
            photopeaInProgress.value = false;
            finalizeSelection();
        }

        async function discardResultImages() {
            if (props.opened) {
                emit('update:opened', false);
            }
            if (photopeaInProgress.value) return;
            photopeaInProgress.value = true;
            await photopeaContext.executeTask(async () => {
                await deselectResultImage();
            });
            photopeaInProgress.value = false;
            finalizeSelection();
        }

        function generateMoreImages() {
            emit('generate-more');
        }

        function generateMoreVariants() {
            // The active image displayed on canvas.
            const displayedImage = selectedResultImages[selectedResultImages.length - 1];
            emit('generate-more-variants', displayedImage);
        }

        const ctrlPressed = ref(false);
        onMounted(() => {
            function onKeydown(e: KeyboardEvent) {
                if (e.key === 'Control') {
                    ctrlPressed.value = true;
                }
            }
            function onKeyup(e: KeyboardEvent) {
                if (e.key === 'Control') {
                    ctrlPressed.value = false;
                }
            }
            window.addEventListener('keydown', onKeydown);
            window.addEventListener('keyup', onKeyup);
        });

        watch(props.images, async (newValue, oldValue) => {
            if (newValue.length === 0) {
                discardResultImages();
                return;
            }
            const imageItem = resultImageItems.value[resultImageItems.value.length - 1];
            if (selectedResultImages.length === 0) {
                // First generation.
                await photopeaContext.executeTask(async () => {
                    if (props.resultDestination == ImageResultDestination.kCurrentCanvas) {
                        await selectResultImage(imageItem);
                    } else { // ImageResultDestination.kNewCanvas
                        await photopeaContext.invoke('pasteImageAsNewDocument', imageItem.imageURL);
                    }
                    selectedResultImages.push(imageItem);
                });
            } else {
                // Generate more.
                await switchResultImage(imageItem);
            }
        });

        function getDrawerWidth() {
            return Math.max(
                (window.innerWidth - 300) * 0.9,
                window.innerWidth - 30
            );
        }
        const drawerWidth = ref<number>(getDrawerWidth());
        onBeforeUpdate(() => {
            drawerWidth.value = getDrawerWidth();
        });
        const onUpdateVisible = (visible: boolean) => {
            emit('update:opened', visible)
        }

        const onGenerateMoreVariants = (event: Event, image: IGeneratedImage) => {
            event.stopPropagation();
            emit('generate-more-variants', image);
        };

        return {
            drawerWidth,
            onUpdateVisible,

            onGenerateMoreVariants,

            resultImageItems,
            selectedResultImageNames,
            photopeaInProgress,

            switchResultImage,
            discardResultImages,
            pickSelectedResultImages,
            generateMoreImages,
            generateMoreVariants,
        };
    },
};
</script>

<template>
    <a-drawer v-bind:visible="opened"
        @update:visible="onUpdateVisible"
        placement="right" v-model:width="drawerWidth"
        title="Generated images">
        <template #closeIcon>
            <ArrowLeftOutlined />
        </template>
        <a-empty description="No generated images"
            v-if="resultImageItems.length === 0"></a-empty>
        <a-spin :spinning="photopeaInProgress">
            <ImagePicker :images="resultImageItems" :selectedImages="selectedResultImageNames" @item-clicked="switchResultImage"
                :actionsOnHover="true" :displayNames="false"
                v-slot="{ image }">
                <div>
                    <a-button size="small"
                        @click="(event: Event) => { onGenerateMoreVariants(event, image); }"
                    >
                        <template #icon>
                            <DiceOutlined />
                        </template>
                    </a-button>
                </div>
            </ImagePicker>
        </a-spin>
        <template #footer>
            <div style="display: flex; gap: 8px;">
                <a-button @click="discardResultImages" danger>
                    <template #icon>
                        <DeleteOutlined />
                    </template>
                </a-button>
                <div style="flex-grow: 1;"></div>
                <a-button @click="generateMoreImages"
                    :title="$t('gen.generateMore')">
                    <template #icon>
                        <RedoOutlined />
                    </template>
                </a-button>
                <a-button @click="pickSelectedResultImages"
                    :disabled="selectedResultImageNames.length === 0"
                    title="Add selected images to canvas">
                    Add to canvas
                </a-button>
            </div>
        </template>
    </a-drawer>
</template>

