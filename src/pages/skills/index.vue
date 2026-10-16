<script setup lang="ts">
import Icon from '@/components/Icon.vue';
import ModalMask from '@/components/ModalMask.vue';
import SummaryOnlyNotice from '@/components/SummaryOnlyNotice.vue';
import { apiSettings } from '@/api/settings';
import { appendOpToLatestLeaf } from '@/memory/apply';
import { derivedMeta, memory } from '@/memory/store';
import type { MemSkill } from '@/memory/types';
import { toast } from '@/st/toast';
import { computed, ref } from 'vue';

const hasLeaf = computed(() => derivedMeta.hasLeaf);
const editing = ref<{ oldName?: string; name: string; desc: string } | null>(null);

function openEditor(skill?: MemSkill) {
  if (!hasLeaf.value || !apiSettings.skillsEnabled) return;
  editing.value = { oldName: skill?.name, name: skill?.name ?? '', desc: skill?.desc ?? '' };
}

function saveSkill() {
  const draft = editing.value;
  if (!draft || !apiSettings.skillsEnabled || !draft.name.trim()) return;
  const name = draft.name.trim();
  const key = name.toLowerCase();
  if (memory.skills.some(s => s.name.toLowerCase() === key && s.name !== draft.oldName)) {
    toast('已有同名技能，请直接编辑已有记录', 'warning');
    return;
  }
  // 改名与编辑一起落到最新叶子，沿用删楼、切换回复时的回退机制。
  const remove = draft.oldName && draft.oldName.toLowerCase() !== key ? [draft.oldName] : undefined;
  if (appendOpToLatestLeaf({ skills: { remove, update: [{ name, desc: draft.desc.trim() }] } })) editing.value = null;
}

function removeSkill(skill: MemSkill) {
  if (apiSettings.skillsEnabled) appendOpToLatestLeaf({ skills: { remove: [skill.name] } });
}
</script>

<template>
  <section class="bbs-page">
    <div class="bbs-skill-head">
      <h2 class="bbs-title bbs-title-sub">技能</h2>
      <button class="bbs-btn bbs-btn-primary" type="button" :disabled="!hasLeaf" @click="openEditor()">添加技能</button>
    </div>
    <SummaryOnlyNotice subject="主角技能与主要角色技能" />
    <p class="bbs-field-hint">记录主角已掌握的技能、效果、等级和使用限制。主要角色的技能请在角色页编辑。{{ hasLeaf ? '后续摘要也会自动维护明确发生的技能变化。' : '需先有摘要才能手动添加。' }}</p>
    <hr class="bbs-rule" />
    <div class="bbs-skill-list">
      <article v-for="skill in memory.skills" :key="skill.id" class="bbs-skill">
        <div class="bbs-skill-head">
          <strong class="bbs-skill-name">{{ skill.name }}</strong>
          <div class="bbs-skill-actions">
            <button class="bbs-btn" type="button" title="编辑技能" @click="openEditor(skill)"><Icon name="edit" /></button>
            <button class="bbs-btn" type="button" title="删除技能" @click="removeSkill(skill)"><Icon name="trash" /></button>
          </div>
        </div>
        <p class="bbs-skill-desc">{{ skill.desc || '暂无技能说明，可点击编辑补充。' }}</p>
      </article>
      <p v-if="!memory.skills.length" class="bbs-field-hint">尚未记录技能。剧情中明确掌握的技能会在摘要后登记，也可手动添加。</p>
    </div>
    <ModalMask :open="!!editing" @close="editing = null">
      <div v-if="editing" class="bbs-modal" role="dialog" aria-modal="true" aria-label="编辑技能">
        <div class="bbs-modal-head">
          <span class="bbs-modal-title">{{ editing.oldName ? '编辑技能' : '添加技能' }}</span>
          <button class="bbs-btn" type="button" aria-label="关闭" @click="editing = null"><Icon name="close" /></button>
        </div>
        <label class="bbs-modal-field">
          <span class="bbs-modal-label">技能名称</span>
          <input v-model="editing.name" class="bbs-input" type="text" placeholder="技能名" />
        </label>
        <label class="bbs-modal-field">
          <span class="bbs-modal-label">技能说明</span>
          <textarea v-model="editing.desc" v-autosize class="bbs-input bbs-modal-textarea" rows="4" placeholder="效果、等级、消耗及使用限制等"></textarea>
        </label>
        <footer class="bbs-modal-foot">
          <button class="bbs-btn" type="button" @click="editing = null">取消</button>
          <button class="bbs-btn bbs-btn-primary" type="button" :disabled="!editing.name.trim()" @click="saveSkill">保存</button>
        </footer>
      </div>
    </ModalMask>
  </section>
</template>

<style scoped>
.bbs-field-hint { margin: 12px 0; font-size: 12px; color: var(--bbs-ink-muted); line-height: 1.6; }
.bbs-modal-textarea { resize: vertical; min-height: 60px; font-family: inherit; }
.bbs-skill-head { display: flex; align-items: center; justify-content: space-between; gap: 12px; }
.bbs-skill-actions { display: flex; flex-shrink: 0; gap: 6px; }
.bbs-skill-list { display: grid; gap: 12px; }
.bbs-skill { padding: 14px; border: 1px solid var(--bbs-line); border-radius: var(--bbs-radius-sm); background: var(--bbs-surface-2); }
.bbs-skill-name { min-width: 0; overflow-wrap: anywhere; }
.bbs-skill-desc { margin: 10px 0 0; color: var(--bbs-ink-soft); white-space: pre-wrap; overflow-wrap: anywhere; }
</style>
