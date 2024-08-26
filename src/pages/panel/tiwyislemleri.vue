<script setup>
import { $api } from '@/utils/api';
import { onMounted, ref, watch } from 'vue';

const items = ref([])
const totalItems = ref(0)
const itemsPerPage = ref(10)
const currentPage = ref(1)
const searchTerms = ref({
  username: '',
  level: '',
  oldBalance: '',
  earnedPoints: '',
  currentBalance: '',
  date: ''
})

const headers = [
  { title: "Profil Görseli", key: "profileImage" },
  { title: "Kullanıcı Adı", key: "username" },
  { title: "Seviye", key: "level" },
  { title: "Eski Puan Bakiyesi", key: "oldBalance" },
  { title: "Kazandığı Puan", key: "earnedPoints" },
  { title: "Güncel Puan Bakiyesi", key: "currentBalance" },
  { title: "Tarih", key: "date" },
  { title: "İşlemler", key: "actions" }
]

const fetchData = async ({ page, itemsPerPage, searchTerms }) => {
  try {
    const skip = (page - 1) * itemsPerPage
    const take = itemsPerPage
    const filters = Object.entries(searchTerms)
      .filter(([, value]) => value)
      .map(([key, value]) => `["${key}","contains","${value}"]`)

    const filterQuery = `filter=[${filters.join(',')}]`

    const { data, totalCount } = await $api(`/system/v1/tiwy?take=${take}&skip=${skip}&${filterQuery}`)

    items.value = data.map(item => ({
      id: item.id,
      profileImage: item.user.profileImage,
      username: item.user.username,
      level: item.user.level,
      oldBalance: item.oldBalance,
      earnedPoints: item.earnedPoints,
      currentBalance: item.currentBalance,
      date: new Date(item.date).toLocaleString('tr-TR', {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
        hour: 'numeric',
        minute: 'numeric',
        hour12: false,
      }),
    }))
    totalItems.value = totalCount
  } catch (error) {
    console.error('Error:', error)
  }
}

const stats = ref({
  current: 0,
  used: 0,
  distributed: 0,
  users: 0,
  total: 0,
  treasury: 0
})

const fetchStats = async () => {
  try {
    const data = await $api('/system/v1/tiwy-stats')
    stats.value = data
  } catch (error) {
    console.error('Error fetching stats:', error)
  }
}

const selectedDateRange = ref([])
const filterType = ref('all')

watch([currentPage, itemsPerPage, searchTerms, selectedDateRange, filterType], () => {
  fetchData({ 
    page: currentPage.value, 
    itemsPerPage: itemsPerPage.value, 
    searchTerms: searchTerms.value,
    dateRange: selectedDateRange.value,
    filterType: filterType.value
  })
  fetchStats()
})

onMounted(() => {
  fetchData({ 
    page: currentPage.value, 
    itemsPerPage: itemsPerPage.value, 
    searchTerms: searchTerms.value,
    dateRange: selectedDateRange.value,
    filterType: filterType.value
  })
  fetchStats()
})

const showUserDetails = (username) => {
  // Implement user details page navigation
  console.log('Show user details for:', username)
}

const editBalance = (item) => {
  // Implement edit balance functionality with 2FA
  console.log('Edit balance for:', item)
}

const showEditHistory = (item) => {
  // Implement show edit history functionality
  console.log('Show edit history for:', item)
}
</script>

<template>
  <div>
    <VCard class="mb-6" title="Tiwy İşlemleri">
      <VCardText>Tiwy ekosistemi ile ilgili verileri takip edin.</VCardText>
    </VCard>

    <VRow>
      <VCol v-for="(value, key) in stats" :key="key" cols="12" sm="6" md="4" lg="2">
        <VCard>
          <VCardTitle>{{ key }}</VCardTitle>
          <VCardText>{{ value }}</VCardText>
        </VCard>
      </VCol>
    </VRow>

    <VCard class="mt-6">
      <VCardTitle>Filtreler</VCardTitle>
      <VCardText>
        <VRow>
          <VCol cols="12" md="6">
            <VSelect
              v-model="filterType"
              :items="['Tüm Zamanlar', 'Özel Tarih Aralığı']"
              label="Filtre Tipi"
            />
          </VCol>
          <VCol cols="12" md="6">
            <VDatePicker
              v-if="filterType === 'Özel Tarih Aralığı'"
              v-model="selectedDateRange"
              range
            />
          </VCol>
        </VRow>
      </VCardText>
    </VCard>

    <VDataTableServer
      v-model:items-per-page="itemsPerPage"
      v-model:page="currentPage"
      :items="items"
      :headers="headers"
      :items-length="totalItems"
      @update:options="fetchData"
    >
      <template #item.profileImage="{ item }">
        <VAvatar>
          <VImg :src="item.profileImage" />
        </VAvatar>
      </template>

      <template #item.username="{ item }">
        <VBtn text @click="showUserDetails(item.username)">
          {{ item.username }}
        </VBtn>
      </template>

      <template #item.actions="{ item }">
        <VBtn icon="mdi-pencil" size="small" @click="editBalance(item)" />
        <VBtn v-if="item.edited" icon="mdi-information" size="small" @click="showEditHistory(item)" />
      </template>
    </VDataTableServer>
  </div>
</template>
