<script lang="ts">
export default {
    name: 'PayloadRadio',
    props: {
        value: {
            type: Number,
            required: true,
        },
        enumType: {
            type: Object,
            required: true,
        },
        label: {
            type: String,
            required: false,
        },
    },
    emits: ['update:value'],
    setup(props, { emit }) {
        return {
            onRadioChange(e: Event) {
                const target = e.target as HTMLInputElement;
                emit('update:value', target.value);
            }
        };
    },
};
</script>

<template>
    <a-space direction="vertical">
        <a-tag style="border: none;" v-if="!!$props.label" class="label">{{ $props.label }}</a-tag>
        <a-radio-group :value="$props.value" @change="onRadioChange" size="small">
            <a-radio-button v-for="key in Object.values($props.enumType).filter(value => isNaN(Number(value)))" :key="key"
                :value="$props.enumType[key]">
                {{ key }}
            </a-radio-button>
        </a-radio-group>
    </a-space>
</template>

<style scoped>
.label {
    border: none;
}
</style>