<template>
    <div v-if="items.length > 0" class="storage-usage" :class="{'storage-usage--multiple': items.length > 1}">
        <div class="storage-usage__item" v-for="item in items" :key="item.label">
            <!-- 録画フォルダが複数のドライブにまたがるときだけ、ドライブ名を表示する -->
            <div class="storage-usage__label" v-if="items.length > 1">{{item.label}}</div>
            <div class="storage-usage__bar" role="progressbar" :aria-label="`${item.label} の使用量`"
                aria-valuemin="0" aria-valuemax="100" :aria-valuenow="Math.round(item.usedPercent)">
                <div class="storage-usage__bar-used" :class="`storage-usage__bar-used--${item.level}`"
                    :style="{width: `${item.usedPercent}%`}"></div>
            </div>
            <div class="storage-usage__text">{{item.text}}</div>
        </div>
    </div>
</template>
<script lang="ts" setup>

import { computed, onMounted, ref } from 'vue';

import Videos, { type IVideoStorageUsage } from '@/services/Videos';
import Utils from '@/utils';

// 録画フォルダがあるドライブ (ボリューム) ごとのストレージ容量
const storages = ref<IVideoStorageUsage[]>([]);

// 空き容量の割合がこの値を下回ったら、バーの色を警告色・危険色に変える
const WARNING_FREE_RATIO = 0.15;
const CRITICAL_FREE_RATIO = 0.05;

// 容量を「123.4 GB」のような、数値と単位の間に空白を入れた表記にする
const formatSize = (bytes: number) => Utils.formatBytes(bytes, 1).replace(/(\d)([A-Z])/, '$1 $2');

// 表示用のデータ
// バーは使用量 (埋まっている部分が使用済み) 、文字は「空き容量 / 全体容量(空き容量の割合)」を表す
const items = computed(() => storages.value.map((storage) => {
    const freeRatio = storage.total > 0 ? Math.min(Math.max(storage.free / storage.total, 0), 1) : 0;
    const level = freeRatio < CRITICAL_FREE_RATIO ? 'critical' : (freeRatio < WARNING_FREE_RATIO ? 'warning' : 'normal');
    return {
        label: storage.label,
        usedPercent: (1 - freeRatio) * 100,
        level,
        text: `${formatSize(storage.free)} / ${formatSize(storage.total)}(${Math.round(freeRatio * 100)}%)`,
    };
}));

onMounted(async () => {
    // 取得に失敗したときは何も表示しない (容量の表示は補助的な情報のため、エラーメッセージも出さない)
    storages.value = await Videos.fetchStorageUsages() ?? [];
});

</script>
<style lang="scss" scoped>

.storage-usage {
    // 複数のドライブを並べるときも、バーの開始位置・終了位置が揃うように、列を揃えて配置する
    display: grid;
    grid-template-columns: minmax(0, 1fr) max-content;
    align-items: center;
    gap: 6px 12px;
    max-width: 640px;
    // 見出しの下の余白 (20px) を少し詰めて、見出しとのまとまりを作る
    margin-top: -8px;
    margin-bottom: 14px;
    @include smartphone-vertical {
        gap: 6px 10px;
        margin-top: -6px;
        margin-bottom: 12px;
        padding: 0px 8px;
    }

    // ドライブ名がある (複数ドライブの) ときは、ドライブ名・バー・文字の3列にする
    &--multiple {
        grid-template-columns: max-content minmax(0, 1fr) max-content;
    }

    // 1ドライブ分の要素 (ドライブ名・バー・文字) を、親のグリッドの列にそのまま配置する
    &__item {
        display: contents;
    }

    &__label {
        flex-shrink: 0;
        font-size: 12px;
        font-weight: 700;
        color: rgb(var(--v-theme-text-darken-1));
    }

    &__bar {
        flex-grow: 1;
        min-width: 0;  // magic!
        height: 8px;
        border-radius: 4px;
        background: rgb(var(--v-theme-background-lighten-2));
        overflow: hidden;

        &-used {
            height: 100%;
            border-radius: 4px;
            background: rgb(var(--v-theme-primary));
            transition: width 0.3s ease;

            &--warning {
                background: rgb(var(--v-theme-warning));
            }
            &--critical {
                background: rgb(var(--v-theme-error));
            }
        }
    }

    &__text {
        flex-shrink: 0;
        font-size: 13px;
        font-variant-numeric: tabular-nums;
        color: rgb(var(--v-theme-text-darken-1));
        white-space: nowrap;
        @include smartphone-vertical {
            font-size: 12px;
        }
    }
}

</style>
