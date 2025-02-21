<script lang="ts">
import { computed, ref } from 'vue';
import { type ILoRA, type IEmbedding } from '../Automatic1111';
import { useA1111ContextStore } from '@/stores/a1111ContextStore';
import { DeploymentUnitOutlined } from '@ant-design/icons-vue';
import ImagePicker from './ImagePicker.vue';

enum NetworkType {
    LoRA = 'LoRA',
    Embedding = 'Embedding',
    Hypernetwork = 'Hypernetwork',
    LyCORIS = 'LyCORIS',
};

interface ImageItem {
    imageURL: string;
    name: string;
};

export default {
    name: 'ExtraNetworks',
    props: {
        loras: {
            type: Array<ILoRA>,
            required: true,
        },
        embeddings: {
            type: Array<[string, IEmbedding]>,
            required: true,
        },
        disabled: {
            type: Boolean,
            default: false
        },
    },
    components: {
        ImagePicker,
        DeploymentUnitOutlined,
    },
    emits: ['add:prompt'],
    setup(props, { emit }) {
        const visible = ref<boolean>(false);
        const currentNetworkType = ref<string>(NetworkType.LoRA);
        const searchKeyword = ref<string>('');

        const loraImages = computed(() => {
            const context = useA1111ContextStore().a1111Context;
            return props.loras
                .map(lora => {
                    return { imageURL: context.loraPreviewURL(lora.name), name: lora.name };
                })
                .filter(item => item.name.toLowerCase().includes(searchKeyword.value.toLowerCase()));
        });

        const embeddingImages = computed(() => {
            const context = useA1111ContextStore().a1111Context;
            return props.embeddings
                .map(entry => {
                    const [name, _] = entry;
                    return { imageURL: context.embeddingPreviewURL(name), name };
                })
                .filter(item => item.name.toLowerCase().includes(searchKeyword.value.toLowerCase()));
        });

        const onAddLoRA = (item: ImageItem) => {
            emit('add:prompt', `<lora:${item.name}:1.0>`);
        };

        const onAddRawPrompt = (item: ImageItem) => {
            emit('add:prompt', `(${item.name}:1.0)`);
        }

        const showDrawer = () => {
            visible.value = true;
        };

        return {
            visible,
            NetworkType,
            currentNetworkType,
            searchKeyword,
            showDrawer,
            loraImages,
            embeddingImages,
            onAddLoRA,
            onAddRawPrompt,
        };
    },
};
</script>

<template>
    <a-button type="primary" @click="showDrawer" :title="$t('gen.extraNetworks')" :disabled="disabled">
        <DeploymentUnitOutlined />
    </a-button>

    <a-drawer v-model:visible="visible" title="Extra Networks" placement="right">
        <a-space direction="vertical">
            <a-radio-group v-model:value="currentNetworkType" button-style="solid">
                <a-radio-button v-for="network in [NetworkType.LoRA, NetworkType.Embedding]" :value="network"
                    :key="network">{{ network
                    }}</a-radio-button>
            </a-radio-group>
            <a-input v-model:value="searchKeyword" style="width: 100%" :placeholder="$t('gen.search') + '...'" />

            <ImagePicker :hidden="currentNetworkType !== NetworkType.LoRA" :images="loraImages" @item-clicked="onAddLoRA">
            </ImagePicker>

            <ImagePicker :hidden="currentNetworkType !== NetworkType.Embedding" :images="embeddingImages"
                @item-clicked="onAddRawPrompt">
            </ImagePicker>
        </a-space>
    </a-drawer>
</template>
