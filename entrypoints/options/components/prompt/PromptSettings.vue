<template>
  <div class="space-y-6">
    <!-- 页面标题 -->
    <div class="border-b border-border pb-4">
      <h2 class="text-2xl font-bold text-foreground">提示词预览</h2>
      <p class="text-sm text-muted-foreground mt-2">
        查看基于您当前设置生成的翻译提示词
      </p>
    </div>

    <!-- 当前设置信息 -->
    <div class="bg-muted p-4 rounded-lg border border-border">
      <h3 class="text-sm font-semibold text-foreground mb-3">当前设置</h3>
      <div class="grid grid-cols-2 md:grid-cols-4 gap-3 text-sm">
        <div>
          <span class="text-muted-foreground">用户水平:</span>
          <span class="ml-2 font-medium">{{ userLevelLabel }}</span>
        </div>
        <div>
          <span class="text-muted-foreground">替换比例:</span>
          <span class="ml-2 font-medium">{{ Math.round(userSettings.replacementRate * 100) }}%</span>
        </div>
        <div>
          <span class="text-muted-foreground">翻译方向:</span>
          <span class="ml-2 font-medium">{{ translationDirectionLabel }}</span>
        </div>
        <div>
          <span class="text-muted-foreground">目标语言:</span>
          <span class="ml-2 font-medium">{{ targetLanguageLabel }}</span>
        </div>
      </div>
    </div>

    <!-- 用户水平调整提示词自定义 -->
    <div class="space-y-4">
      <div class="flex items-center justify-between">
        <h3 class="text-lg font-semibold text-foreground">用户水平调整提示词</h3>
        <div class="flex items-center space-x-2">
          <button
            @click="resetLevelAdjustment"
            class="px-3 py-2 text-sm bg-secondary text-secondary-foreground rounded-md hover:bg-secondary/80"
          >
            重置为默认
          </button>
          <button
            @click="saveLevelAdjustment"
            class="px-3 py-2 text-sm bg-primary text-primary-foreground rounded-md hover:bg-primary/90"
          >
            保存设置
          </button>
        </div>
      </div>

      <div class="bg-blue-50 dark:bg-blue-900/20 p-3 rounded-lg border border-blue-200 dark:border-blue-800">
        <p class="text-sm text-blue-600 dark:text-blue-400">
          <strong>💡 提示：</strong> 使用 <code class="bg-blue-100 dark:bg-blue-800 px-1 py-0.5 rounded text-xs">${UserLevel}</code> 作为占位符，系统会自动替换为当前用户的水平等级
        </p>
      </div>

      <!-- 当前生成的内容预览 -->
      <div class="space-y-2">
        <h4 class="text-sm font-medium text-foreground">当前生成内容</h4>
        <div class="bg-gray-50 dark:bg-gray-800 p-3 rounded-lg border border-gray-200 dark:border-gray-700">
          <code class="text-sm text-gray-600 dark:text-gray-300">{{ currentLevelAdjustment }}</code>
        </div>
      </div>

      <!-- 自定义编辑器 -->
      <div class="space-y-2">
        <h4 class="text-sm font-medium text-foreground">自定义模板</h4>
        <textarea
          v-model="customLevelAdjustment"
          rows="4"
          class="w-full px-3 py-2 border border-border rounded-md focus:ring-2 focus:ring-primary focus:border-primary font-mono text-sm"
          placeholder="例如：The user's proficiency is at the ${UserLevel} level. Please adjust the difficulty and frequency of the selected words accordingly."
        />
      </div>


    </div>

    <!-- 提示词预览 -->
    <div class="space-y-4">
      <div class="flex items-center justify-between">
        <h3 class="text-lg font-semibold text-foreground">生成的提示词</h3>
        <button
          @click="refreshPrompt"
          class="px-3 py-2 text-sm bg-secondary text-secondary-foreground rounded-md hover:bg-secondary/80"
          :disabled="loading"
        >
          {{ loading ? '加载中...' : '刷新' }}
        </button>
      </div>

      <div class="space-y-2">
        <div class="bg-muted p-4 rounded-lg border border-border max-h-96 overflow-y-auto">
          <pre v-if="!loading" class="text-sm text-muted-foreground whitespace-pre-wrap font-mono">{{ currentPrompt }}</pre>
          <div v-else class="text-sm text-muted-foreground text-center py-8">
            正在加载用户设置...
          </div>
        </div>
      </div>

      <!-- 统计信息 -->
      <div v-if="!loading" class="bg-blue-50 dark:bg-blue-900/20 p-3 rounded-lg border border-blue-200 dark:border-blue-800">
        <div class="flex items-center space-x-4 text-sm">
          <div>
            <span class="text-blue-600 dark:text-blue-400">提示词长度:</span>
            <span class="ml-1 font-medium">{{ currentPrompt.length }} 字符</span>
          </div>
          <div>
            <span class="text-blue-600 dark:text-blue-400">预计Token数:</span>
            <span class="ml-1 font-medium">约 {{ Math.ceil(currentPrompt.length / 4) }} tokens</span>
          </div>
        </div>
      </div>


    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import { promptService, PromptService } from '../../../../src/modules/core/translation/PromptService';
import { storageService } from '../../../../src/modules/core/storage/StorageService';
import { UserLevel, USER_LEVEL_OPTIONS } from '../../../../src/modules/shared/types/core';
import type { PromptConfig } from '../../../../src/modules/core/translation/types';
import type { UserSettings } from '../../../../src/modules/shared/types/storage';

const emit = defineEmits<{
  saveMessage: [message: string];
}>();

// 响应式状态
const userSettings = ref<UserSettings>({} as UserSettings);
const loading = ref(true);

// 用户水平调整提示词相关
const defaultLevelAdjustment = PromptService.DEFAULT_LEVEL_ADJUSTMENT_TEMPLATE;
const customLevelAdjustment = ref(defaultLevelAdjustment);
const savedCustomLevelAdjustment = ref<string>(''); // 保存用户自定义的版本

// 计算属性
const userLevelLabel = computed(() => {
  const userLevel = userSettings.value.userLevel;
  
  // 如果是预定义等级
  if (typeof userLevel === 'number') {
    const option = USER_LEVEL_OPTIONS.find(opt => opt.value === userLevel);
    return option ? option.label : '未知';
  }
  
  // 如果是自定义等级
  if (typeof userLevel === 'string') {
    const customLevel = userSettings.value.customUserLevels?.find(level => level.id === userLevel);
    return customLevel ? customLevel.name : userLevel;
  }
  
  return '未知';
});

const translationDirectionLabel = computed(() => {
  const direction = userSettings.value.translationDirection;
  if (userSettings.value.multilingualConfig?.intelligentMode) {
    return '智能模式';
  }
  
  switch (direction) {
    case 'zh-to-en': return '中文→英语';
    case 'en-to-zh': return '英语→中文';
    case 'ja-to-zh': return '日语→中文';
    case 'ko-to-zh': return '韩语→中文';
    default: return direction || '未知';
  }
});

const targetLanguageLabel = computed(() => {
  const targetLang = userSettings.value.multilingualConfig?.targetLanguage || 'en';
  switch (targetLang) {
    case 'en': return '英语';
    case 'zh': return '中文';
    case 'ja': return '日语';
    case 'ko': return '韩语';
    default: return targetLang;
  }
});

// 生成提示词配置
const promptConfig = computed<PromptConfig>(() => ({
  targetLanguage: userSettings.value.multilingualConfig?.targetLanguage || 'en',
  translationDirection: userSettings.value.multilingualConfig?.intelligentMode 
    ? 'intelligent' 
    : userSettings.value.translationDirection || 'intelligent',
  userLevel: userSettings.value.userLevel || UserLevel.INTERMEDIATE,
  replacementRate: userSettings.value.replacementRate || 0.3,
  provider: 'openai',
  intelligentMode: userSettings.value.multilingualConfig?.intelligentMode || false,
  customLevelAdjustment: userSettings.value.customLevelAdjustment || undefined,
  customUserLevels: userSettings.value.customUserLevels?.map(level => ({
    id: level.id,
    name: level.name
  })) || [],
}));

// 计算当前提示词
const currentPrompt = computed(() => {
  if (loading.value) {
    return '';
  }
  return promptService.getUnifiedPrompt(promptConfig.value);
});

// 用户水平调整相关计算属性
const currentLevelAdjustment = computed(() => {
  if (loading.value) return '';
  const template = savedCustomLevelAdjustment.value || defaultLevelAdjustment;
  return template.replace(/\$\{UserLevel\}/g, userLevelLabel.value);
});





// 刷新提示词
const refreshPrompt = async () => {
  await loadUserSettings();
};

// 用户水平调整相关功能

const resetLevelAdjustment = async () => {
  try {
    customLevelAdjustment.value = defaultLevelAdjustment;
    
    // 从存储中删除自定义设置
    const currentSettings = await storageService.getUserSettings();
    currentSettings.customLevelAdjustment = undefined;
    
    const result = await storageService.saveUserSettings(currentSettings);
    
    if (result.success) {
      savedCustomLevelAdjustment.value = '';
      emit('saveMessage', '已重置为默认模板');
    } else {
      emit('saveMessage', '重置失败：' + result.error);
    }
  } catch (error) {
    console.error('Failed to reset level adjustment:', error);
    emit('saveMessage', '重置失败，请重试');
  }
};

const saveLevelAdjustment = async () => {
  try {
    // 更新用户设置
    const currentSettings = await storageService.getUserSettings();
    currentSettings.customLevelAdjustment = customLevelAdjustment.value === defaultLevelAdjustment ? undefined : customLevelAdjustment.value;
    
    // 保存到存储中
    const result = await storageService.saveUserSettings(currentSettings);
    
    if (result.success) {
      savedCustomLevelAdjustment.value = customLevelAdjustment.value === defaultLevelAdjustment ? '' : customLevelAdjustment.value;
      emit('saveMessage', '用户水平调整提示词已保存');
    } else {
      emit('saveMessage', '保存失败：' + result.error);
    }
  } catch (error) {
    console.error('Failed to save level adjustment:', error);
    emit('saveMessage', '保存失败，请重试');
  }
};

// 加载用户设置
const loadUserSettings = async () => {
  try {
    loading.value = true;
    userSettings.value = await storageService.getUserSettings();
    
    // 更新自定义提示词状态
    const customSetting = userSettings.value.customLevelAdjustment;
    savedCustomLevelAdjustment.value = customSetting || '';
    customLevelAdjustment.value = customSetting || defaultLevelAdjustment;
  } catch (error) {
    console.error('Failed to load user settings:', error);
    emit('saveMessage', '加载用户设置失败');
  } finally {
    loading.value = false;
  }
};

// 组件挂载时加载设置
onMounted(async () => {
  await loadUserSettings();
});
</script> 