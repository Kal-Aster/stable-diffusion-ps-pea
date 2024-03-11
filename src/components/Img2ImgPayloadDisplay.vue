<script lang="ts">
import {
    Img2ImgPayload, ResizeMode,
    InpaintArea,
    InpaintFill,
    MaskMode,
} from '@/Automatic1111';
import PayloadRadio from '@/components/PayloadRadio.vue';

export default {
    name: 'Img2ImgPayloadDisplay',
    props: {
        payload: {
            type: Img2ImgPayload,
            required: true,
        },
    },
    components: {
        PayloadRadio,
    },
    setup(props) {
        return {
            ResizeMode,
            InpaintArea,
            InpaintFill,
            MaskMode,

            updateMaskBlur(value: number) {
                props.payload.mask_blur_x = value;
                props.payload.mask_blur_y = value;
                props.payload.mask_blur = value;
            }
        };
    },
};
</script>

<template>
    <a-space direction="vertical" size="middle">
        <PayloadRadio :label="$t('gen.resizeMode')" v-model:value="payload.resize_mode" :enum-type="ResizeMode">
        </PayloadRadio>
        <PayloadRadio :label="$t('gen.inpaintArea')" v-model:value="payload.inpaint_full_res" :enum-type="InpaintArea">
        </PayloadRadio>
        <PayloadRadio :label="$t('gen.inpaintFill')" v-model:value="payload.inpainting_fill" :enum-type="InpaintFill">
        </PayloadRadio>
        <PayloadRadio :label="$t('gen.maskMode')" v-model:value="payload.inpainting_mask_invert" :enum-type="MaskMode">
        </PayloadRadio>
        <a-space direction="vertical">
            <a-tag style="border: none;">Only masked padding</a-tag>
            <a-input-number addonAfter="px" v-model:value="payload.inpaint_full_res_padding" :min="0" />
        </a-space>
        <a-space direction="vertical">
            <a-tag style="border: none;">Mask blur</a-tag>
            <a-input-number addonAfter="px" :value="payload.mask_blur_x" :min="0" @change="updateMaskBlur" />
        </a-space>
    </a-space>
</template>
