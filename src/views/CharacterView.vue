<script setup lang="ts">
import { computed, ref } from 'vue'
import { useInventoryStore } from '../stores/inventory'
import type { EquipmentSlot, InventoryItem } from '../types/inventory'

const store = useInventoryStore()
const dragged = ref<string | null>(null)
const message = ref('')
const slots: EquipmentSlot[] = ['head', 'body', 'armor', 'weapon', 'backpack']
const labels: Record<EquipmentSlot, string> = {
  head: 'Голова', body: 'Одежда', armor: 'Броня', weapon: 'Оружие', backpack: 'Рюкзак'
}
const equipped = (slot: EquipmentSlot) => store.items.find(item => item.id === store.equipment[slot])
const available = computed(() => store.items.filter(item => slots.some(slot => store.canEquip(item, slot))))
const totalWeight = computed(() => store.items.reduce((sum, item) => sum + item.weight * item.quantity, 0))
const protection = computed(() => slots.filter(slot => ['head', 'body', 'armor'].includes(slot)).reduce((sum, slot) => sum + (equipped(slot)?.type === 'armor' ? 20 : 0), 0))
const backpackBonus = computed(() => equipped('backpack') ? 20 : 0)
const maxWeight = computed(() => 35 + backpackBonus.value)
const load = computed(() => Math.min(100, Math.round(totalWeight.value / maxWeight.value * 100)))

function equip(slot: EquipmentSlot, id: string) {
  if (!id) { store.unequip(slot); message.value = ''; return }
  message.value = store.equip(id, slot) ? `${labels[slot]}: предмет экипирован` : 'Предмет не подходит для этого слота'
}
function drop(slot: EquipmentSlot) {
  if (dragged.value) equip(slot, dragged.value)
  dragged.value = null
}
</script>

<template>
  <section class="page character-page">
    <p class="eyebrow">PROFILE / 02</p>
    <h1>ПЕРСОНАЖ</h1>
    <p class="muted">Экипируй предметы перетаскиванием или через меню слота. Все изменения сохраняются вместе с инвентарём.</p>
    <div class="character-dashboard">
      <div class="panel character-main">
        <div class="character-topline"><strong>ОПЕРАТОР // B1ADEDEV</strong><span>● ONLINE</span></div>
        <p v-if="message" class="equip-message" role="status">{{ message }}</p>
        <div class="character-layout">
          <div class="equipment-zone">
            <div v-for="slot in slots.slice(0, 2)" :key="slot" class="equip" :class="{filled: equipped(slot)}"
              @dragover.prevent @drop.prevent="drop(slot)">
              <span>{{ labels[slot] }}</span><div class="equipped-icon">{{ equipped(slot)?.icon || '＋' }}</div>
              <b>{{ equipped(slot)?.name || 'Пустой слот' }}</b>
              <select :aria-label="labels[slot]" :value="store.equipment[slot] || ''" @change="equip(slot, ($event.target as HTMLSelectElement).value)">
                <option value="">Снять</option>
                <option v-for="item in store.items.filter(item => store.canEquip(item, slot) || store.equipment[slot] === item.id)" :key="item.id" :value="item.id">{{ item.name }}</option>
              </select>
            </div>
          </div>
          <div class="character-figure" aria-label="Силуэт персонажа">
            <div class="figure-halo"></div>
            <div class="figure-head"></div><div class="figure-neck"></div>
            <div class="figure-torso"></div><div class="figure-arm left"></div><div class="figure-arm right"></div>
            <div class="figure-leg left"></div><div class="figure-leg right"></div>
            <span v-if="equipped('head')" class="figure-gear helmet">{{ equipped('head')?.icon }}</span>
            <span v-if="equipped('armor')" class="figure-gear vest">{{ equipped('armor')?.icon }}</span>
            <span v-if="equipped('weapon')" class="figure-gear weapon">{{ equipped('weapon')?.icon }}</span>
            <span v-if="equipped('backpack')" class="figure-gear bag">{{ equipped('backpack')?.icon }}</span>
          </div>
          <div class="equipment-zone">
            <div v-for="slot in slots.slice(2)" :key="slot" class="equip" :class="{filled: equipped(slot)}"
              @dragover.prevent @drop.prevent="drop(slot)">
              <span>{{ labels[slot] }}</span><div class="equipped-icon">{{ equipped(slot)?.icon || '＋' }}</div>
              <b>{{ equipped(slot)?.name || 'Пустой слот' }}</b>
              <select :aria-label="labels[slot]" :value="store.equipment[slot] || ''" @change="equip(slot, ($event.target as HTMLSelectElement).value)">
                <option value="">Снять</option>
                <option v-for="item in store.items.filter(item => store.canEquip(item, slot) || store.equipment[slot] === item.id)" :key="item.id" :value="item.id">{{ item.name }}</option>
              </select>
            </div>
          </div>
        </div>
      </div>
      <aside class="character-sidebar">
        <div class="panel character-stats">
          <h2>ХАРАКТЕРИСТИКИ</h2>
          <div class="stat-line"><span>Здоровье</span><strong>100 / 100</strong></div><div class="stat-track"><div style="width:100%"></div></div>
          <div class="stat-line"><span>Выносливость</span><strong>100 / 100</strong></div><div class="stat-track"><div style="width:100%"></div></div>
          <div class="stat-line"><span>Защита экипировки</span><strong>{{ protection }}</strong></div><div class="stat-track"><div :style="{width: Math.min(protection, 100) + '%'}"></div></div>
          <div class="stat-line"><span>Нагрузка</span><strong>{{ totalWeight.toFixed(1) }} / {{ maxWeight }} кг</strong></div>
          <div class="stat-track"><div :class="{overloaded: totalWeight > maxWeight}" :style="{width: load + '%'}"></div></div>
          <p v-if="totalWeight > maxWeight" class="equip-message">Превышен рекомендуемый вес!</p>
          <p class="muted">Здоровье и выносливость пока отображаются как базовые показатели; защита и нагрузка рассчитываются по предметам.</p>
        </div>
        <div class="panel character-items">
          <h2>ДОСТУПНАЯ ЭКИПИРОВКА</h2>
          <p class="muted">Перетащи предмет в подходящий слот.</p>
          <div v-for="item in available" :key="item.id" class="character-item" draggable="true"
            @dragstart="dragged = item.id" @dragend="dragged = null">
            <span class="item-emoji">{{ item.icon }}</span>
            <div><strong>{{ item.name }}</strong><small>{{ item.type }} · {{ item.weight }} кг</small></div>
            <span class="drag-handle">⠿</span>
          </div>
          <p v-if="!available.length" class="muted">Нет подходящих предметов. Добавь их в инвентарь.</p>
        </div>
      </aside>
    </div>
  </section>
</template>
