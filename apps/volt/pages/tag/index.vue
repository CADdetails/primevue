<template>
    <DocComponent title="Vue Tag Component" header="Tag" description="Tag component is used to categorize content." :componentDocs="docs" />
</template>

<script setup lang="ts">
import BasicDoc from '@/doc/tag/BasicDoc.vue';
import DownloadDoc from '@/doc/tag/DownloadDoc.vue';
import IconDoc from '@/doc/tag/IconDoc.vue';
import ImportDoc from '@/doc/tag/ImportDoc.vue';
import PillDoc from '@/doc/tag/PillDoc.vue';
import SeverityDoc from '@/doc/tag/SeverityDoc.vue';
import TemplateDoc from '@/doc/tag/TemplateDoc.vue';
import { ref } from 'vue';

const docs = ref([
    {
        id: 'dowload',
        label: 'Download',
        component: DownloadDoc
    },
    {
        id: 'import',
        label: 'Import',
        component: ImportDoc
    },
    {
        id: 'basic',
        label: 'Basic',
        component: BasicDoc
    },
    {
        id: 'severity',
        label: 'Severity',
        component: SeverityDoc
    },
    {
        id: 'pill',
        label: 'Pill',
        component: PillDoc
    },
    {
        id: 'icons',
        label: 'Icon',
        component: IconDoc
    },
    {
        id: 'template',
        label: 'Template',
        component: TemplateDoc
    }
]);
</script>
