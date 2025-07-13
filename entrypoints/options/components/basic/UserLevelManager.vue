<template>
  <div class="space-y-4">
    <!-- 当前等级选择 -->
    <div class="space-y-2">
      <Label for="current-level">当前用户等级</Label>
      <Select
        :model-value="currentLevel"
        @update:model-value="onLevelChange"
      >
        <SelectTrigger>
          <SelectValue placeholder="选择用户等级" />
        </SelectTrigger>
        <SelectContent>
          <!-- 预定义等级 -->
          <SelectGroup>
            <SelectLabel>预定义等级</SelectLabel>
            <SelectItem
              v-for="level in predefinedLevels"
              :key="level.value"
              :value="level.value.toString()"
            >
              {{ level.label }}
            </SelectItem>
          </SelectGroup>
          <!-- 自定义等级 -->
          <SelectGroup v-if="customLevels && customLevels.length > 0">
            <SelectLabel>自定义等级</SelectLabel>
            <SelectItem
              v-for="level in customLevels"
              :key="level.id"
              :value="level.id"
            >
              {{ level.name }}
              <span v-if="level.description" class="text-muted-foreground ml-1">
                ({{ level.description }})
              </span>
            </SelectItem>
          </SelectGroup>
        </SelectContent>
      </Select>
    </div>

    <!-- 自定义等级管理 -->
    <div class="border-t border-border pt-4">
      <div class="flex items-center justify-between mb-3">
        <h4 class="text-sm font-medium text-foreground">自定义等级管理</h4>
        <Button
          @click="showAddDialog = true"
          size="sm"
          variant="outline"
        >
          添加等级
        </Button>
      </div>

      <!-- 自定义等级列表 -->
      <div v-if="customLevels && customLevels.length > 0" class="space-y-2">
        <div
          v-for="level in customLevels"
          :key="level.id"
          class="flex items-center justify-between p-3 border border-border rounded-md"
        >
          <div class="flex-1">
            <div class="font-medium">{{ level.name }}</div>
            <div v-if="level.description" class="text-sm text-muted-foreground">
              {{ level.description }}
            </div>
          </div>
          <div class="flex items-center space-x-2">
            <Button
              @click="editLevel(level)"
              size="sm"
              variant="ghost"
            >
              编辑
            </Button>
            <Button
              @click="deleteLevel(level.id)"
              size="sm"
              variant="ghost"
              class="text-destructive hover:text-destructive"
            >
              删除
            </Button>
          </div>
        </div>
      </div>

      <div v-else class="text-sm text-muted-foreground text-center py-4">
        暂无自定义等级，点击"添加等级"创建新的等级
      </div>
    </div>

    <!-- 添加/编辑等级对话框 -->
    <div
      v-if="showAddDialog || editingLevel"
      class="fixed inset-0 bg-black/50 flex items-center justify-center z-50"
      @click.self="closeDialog"
    >
      <div class="bg-background p-6 rounded-lg border border-border max-w-md w-full mx-4">
        <h3 class="text-lg font-semibold mb-4">
          {{ editingLevel ? '编辑等级' : '添加新等级' }}
        </h3>
        
        <div class="space-y-4">
          <div>
            <Label for="level-name">等级名称</Label>
            <Input
              id="level-name"
              v-model="formData.name"
              placeholder="例如: CET4, CET6, IELTS 6.5"
              class="mt-1"
            />
          </div>
          
          <div>
            <Label for="level-description">描述 (可选)</Label>
            <Textarea
              id="level-description"
              v-model="formData.description"
              placeholder="等级的详细描述..."
              rows="3"
              class="mt-1"
            />
          </div>
        </div>

        <div class="flex justify-end space-x-2 mt-6">
          <Button
            @click="closeDialog"
            variant="outline"
          >
            取消
          </Button>
          <Button
            @click="saveLevel"
            :disabled="!formData.name.trim()"
          >
            {{ editingLevel ? '保存' : '添加' }}
          </Button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch } from 'vue';
import { USER_LEVEL_OPTIONS, UserLevel, type CustomUserLevel } from '../../../../src/modules/shared/types/core';
import { Label } from '../../../../components/ui/label';
import { Button } from '../../../../components/ui/button';
import { Input } from '../../../../components/ui/input';
import { Textarea } from '../../../../components/ui/textarea';
import {
  Select,
  SelectContent,
  SelectGroup,
  SelectItem,
  SelectLabel,
  SelectTrigger,
  SelectValue,
} from '../../../../components/ui/select';

interface Props {
  currentLevel: UserLevel | string;
  customLevels?: CustomUserLevel[]; // 使用可选属性
}

interface Emits {
  'update:currentLevel': [value: UserLevel | string];
  'update:customLevels': [value: CustomUserLevel[]];
}

const props = withDefaults(defineProps<Props>(), {
  customLevels: () => [] // 提供默认值
});
const emit = defineEmits<Emits>();

// 预定义等级选项
const predefinedLevels = USER_LEVEL_OPTIONS;

// 对话框状态
const showAddDialog = ref(false);
const editingLevel = ref<CustomUserLevel | null>(null);
const formData = ref({
  name: '',
  description: '',
});

// 当前选中的等级
const currentLevel = computed({
  get: () => props.currentLevel,
  set: (value) => emit('update:currentLevel', value),
});

// 自定义等级列表
const customLevels = computed({
  get: () => props.customLevels || [], // 提供默认值
  set: (value) => emit('update:customLevels', value),
});

// 等级改变处理
const onLevelChange = (value: any) => {
  if (!value) return;
  
  const strValue = value.toString();
  
  // 如果是数字，转换为UserLevel枚举
  if (/^\d+$/.test(strValue)) {
    currentLevel.value = parseInt(strValue) as UserLevel;
  } else {
    // 否则是自定义等级ID
    currentLevel.value = strValue;
  }
};

// 编辑等级
const editLevel = (level: CustomUserLevel) => {
  editingLevel.value = level;
  formData.value = {
    name: level.name,
    description: level.description || '',
  };
};

// 删除等级
const deleteLevel = (levelId: string) => {
  // 如果当前选中的是要删除的等级，切换到默认等级
  if (currentLevel.value === levelId) {
    currentLevel.value = UserLevel.INTERMEDIATE;
  }
  
  const newLevels = customLevels.value.filter(level => level.id !== levelId);
  customLevels.value = newLevels;
};

// 保存等级
const saveLevel = () => {
  if (!formData.value.name.trim()) return;

  if (editingLevel.value) {
    // 编辑现有等级
    const index = customLevels.value.findIndex(level => level.id === editingLevel.value!.id);
    if (index !== -1) {
      const updatedLevels = [...customLevels.value];
      updatedLevels[index] = {
        ...editingLevel.value,
        name: formData.value.name.trim(),
        description: formData.value.description.trim() || undefined,
      };
      customLevels.value = updatedLevels;
    }
  } else {
    // 添加新等级
    const newLevel: CustomUserLevel = {
      id: `custom-${Date.now()}`,
      name: formData.value.name.trim(),
      description: formData.value.description.trim() || undefined,
      createdAt: Date.now(),
    };
    customLevels.value = [...customLevels.value, newLevel];
  }

  closeDialog();
};

// 关闭对话框
const closeDialog = () => {
  showAddDialog.value = false;
  editingLevel.value = null;
  formData.value = {
    name: '',
    description: '',
  };
};
</script> 