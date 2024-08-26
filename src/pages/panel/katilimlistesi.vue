<script setup>
import { $api } from '@/utils/api';
import { onMounted, ref, watch } from 'vue';

const items = ref([])
const totalItems = ref(0)
const itemsPerPage = ref(10)
const currentPage = ref(1)
const searchTerms = ref({
  username: '',
  experience: '',
  type: '',
  game: '',
  comment: '',
  odd_val: '',
  status: '',
  created_date: ''
})

const headers = [
  { title: "Kullanıcı", key: "username" },
  { title: "Seviye", key: "experience" },
  { title: "Boost Sayısı", key: "type" },
  { title: "Oyun", key: "game" },
  { title: "Yorum", key: "comment" },
  { title: "Oran", key: "odd_val" },
  { title: "Durum", key: "status" },
  { title: "Oluşturulma Tarihi", key: "created_date" },
  { title: "İşlemler", key: "actions" }
]

const fetchData = async ({ page, itemsPerPage, searchTerms }) => {
  try {
    const skip = (page - 1) * itemsPerPage
    const take = itemsPerPage
    const filters = Object.entries(searchTerms)
      .filter(([, value]) => value)
      .map(([key, value]) => `["${key}","contains","${value}"]`)

    filters.push('["type",">",0]')

    const filterQuery = `filter=[${filters.join(',')}]`

    const { data, totalCount } = await $api(`/system/v1/comments?take=${take}&skip=${skip}&${filterQuery}`)

    items.value = data.map(comment => ({
      id: comment.id,
      username: comment.user.username,
      league: comment.league.name,
      teams: `${comment.team1.name} - ${comment.team2.name}`,
      game: comment.game.name,
      comment: comment.comment,
      created_date: new Date(comment.created_date).toLocaleString('tr-TR', {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
        hour: 'numeric',
        minute: 'numeric',
        hour12: false,
      }),
      view: comment.view,
      odd_val: comment.odd_val,
      status: comment.status,
    }))
    totalItems.value = totalCount
  } catch (error) {
    console.error('Error:', error)
  }
}

watch([currentPage, itemsPerPage, searchTerms], () => {
  fetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value })
})

onMounted(() => {
  fetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value })
})

const editItem = (item) => {
  // Implement edit functionality
  console.log('Edit item:', item)
}

const deleteItem = (item) => {
  // Implement delete functionality
  console.log('Delete item:', item)
}
</script>

<template>
  <div>
    <VCard class="mb-6" title="Katılım Listesi">
      <VCardText>Boost katılımı sağlayan kullanıcıların listesini gösterir.</VCardText>
    </VCard>

    <VDataTableServer
      v-model:items-per-page="itemsPerPage"
      v-model:page="currentPage"
      :items="items"
      :headers="headers"
      :items-length="totalItems"
      @update:options="fetchData"
    >
      <template #header="{ headers }">
        <tr>
          <th v-for="header in headers" :key="header.key">
            <VTextField
              v-model="searchTerms[header.key]"
              :label="header.title"
              density="compact"
              hide-details
              variant="solo"
              @update:model-value="fetchData({ page: currentPage, itemsPerPage, searchTerms })"
            />
          </th>
        </tr>
      </template>

      <template #item.status="{ item }">
        <VChip :color="item.status === 1 ? 'success' : 'error'">
          {{ item.status === 1 ? 'Onaylandı' : 'Pasif' }}
        </VChip>
      </template>

      <template #item.actions="{ item }">
        <VBtn icon="mdi-pencil" size="small" @click="editItem(item)" />
        <VBtn icon="mdi-delete" size="small" @click="deleteItem(item)" />
      </template>
    </VDataTableServer>
  </div>
</template>
