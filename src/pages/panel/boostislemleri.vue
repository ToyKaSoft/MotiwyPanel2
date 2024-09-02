<script setup>
import { $api } from '@/utils/api';
import fetch from 'node-fetch'; // Add this import at the top of the file
import { onMounted, ref, watch } from 'vue';

const items = ref([])
const totalItems = ref(0)
const itemsPerPage = ref(10)
const currentPage = ref(1)
const searchTerms = ref({
  username: '',
  level: '',
  purchasedBoosts: '',
  amount: '',
  usedBoosts: '',
  purchaseDate: ''
})

const headers = [
  { title: "Kullanıcı Adı", key: "username" },
  { title: "Seviye", key: "level" },
  { title: "Satın Alınan Boost Adeti", key: "purchasedBoosts" },
  { title: "Tutar", key: "amount" },
  { title: "Kullanılan Boost Sayısı", key: "usedBoosts" },
  { title: "Satın Alma Tarihi", key: "purchaseDate" },
]

const filterOptions = ref([
  { title: 'Tümü', value: 'all' },
  { title: 'Kullanılan', value: 'used' },
  { title: 'Kullanılmayan', value: 'unused' },
  { title: 'İptal Edilen', value: 'cancelled' },
])

const selectedFilter = ref('all')

const fetchData = async ({ page, itemsPerPage, searchTerms, filter }) => {
  try {
    const skip = (page - 1) * itemsPerPage
    const take = itemsPerPage
    const filters = Object.entries(searchTerms)
      .filter(([, value]) => value)
      .map(([key, value]) => `["${key}","contains","${value}"]`)

    if (filter !== 'all') {
      filters.push(`["status","=","${filter}"]`)
    }

    const filterQuery = `filter=[${filters.join(',')}]`

    const url = `http://127.0.0.1:8011/api/system/v1/comments/?take=${take}&skip=${skip}&${filterQuery}`

    const response = await fetch(url, {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        // Add any necessary authentication headers here
      },
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const { data, totalCount } = await response.json();

    items.value = data.map(boost => ({
      id: boost.id,
      profileImage: boost.user.profileImage,
      username: boost.user.username,
      level: boost.user.level,
      purchasedBoosts: boost.purchasedBoosts,
      amount: boost.amount,
      usedBoosts: boost.usedBoosts,
      purchaseDate: new Date(boost.purchaseDate).toLocaleString('tr-TR', {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
        hour: 'numeric',
        minute: 'numeric',
        hour12: false,
      }),
      status: boost.status,
    }))
    totalItems.value = totalCount
  } catch (error) {
    console.error('Error:', error)
  }
}

watch([currentPage, itemsPerPage, searchTerms, selectedFilter], () => {
  fetchData({ 
    page: currentPage.value, 
    itemsPerPage: itemsPerPage.value, 
    searchTerms: searchTerms.value,
    filter: selectedFilter.value
  })
})

onMounted(() => {
  fetchData({ 
    page: currentPage.value, 
    itemsPerPage: itemsPerPage.value, 
    searchTerms: searchTerms.value,
    filter: selectedFilter.value
  })
})

const showUserDetails = (username) => {
  // Implement user details page navigation
  console.log('Show user details for:', username)
}

const showPromotionModal = () => {
  // Implement promotion modal
  console.log('Show promotion modal')
}

const showPackagesModal = () => {
  // Implement packages modal
  console.log('Show packages modal')
}

const boostStats = ref({
  today: 0,
  thisMonth: 0,
  allTime: 0
})

const fetchBoostStats = async () => {
  try {
    const { today, thisMonth, allTime } = await $api('/system/v1/boost-stats')
    boostStats.value = { today, thisMonth, allTime }
  } catch (error) {
    console.error('Error fetching boost stats:', error)
  }
}

onMounted(fetchBoostStats)
</script>

<template>
  <div>
    <VCard class="mb-6" title="Boost İşlemleri">
      <VCardText>Boost işlemlerini ve istatistiklerini gösterir.</VCardText>
    </VCard>

    <VRow class="mb-6">
      <VCol cols="4">
        <VCard>
          <VCardTitle>Bugün</VCardTitle>
          <VCardText>{{ boostStats.today }} Boost</VCardText>
        </VCard>
      </VCol>
      <VCol cols="4">
        <VCard>
          <VCardTitle>Bu Ay</VCardTitle>
          <VCardText>{{ boostStats.thisMonth }} Boost</VCardText>
        </VCard>
      </VCol>
      <VCol cols="4">
        <VCard>
          <VCardTitle>Tüm Zamanlar</VCardTitle>
          <VCardText>{{ boostStats.allTime }} Boost</VCardText>
        </VCard>
      </VCol>
    </VRow>

    <VRow class="mb-6">
      <VCol>
        <VBtn @click="showPromotionModal">Promosyon</VBtn>
        <VBtn @click="showPackagesModal" class="ml-4">Paketler</VBtn>
      </VCol>
    </VRow>

    <VDataTableServer
      v-model:items-per-page="itemsPerPage"
      v-model:page="currentPage"
      :items="items"
      :headers="headers"
      :items-length="totalItems"
      @update:options="fetchData"
    >
      <template #top>
        <VRow>
          <VCol cols="8">
            <VTextField
              v-model="searchTerms.username"
              label="Ara"
              density="compact"
              hide-details
              variant="solo"
              @update:model-value="fetchData({ page: currentPage, itemsPerPage, searchTerms, filter: selectedFilter })"
            />
          </VCol>
          <VCol cols="4">
            <VSelect
              v-model="selectedFilter"
              :items="filterOptions"
              label="Filtre"
              density="compact"
              hide-details
              variant="solo"
              @update:model-value="fetchData({ page: currentPage, itemsPerPage, searchTerms, filter: selectedFilter })"
            />
          </VCol>
        </VRow>
      </template>

      <template #item.profileImage="{ item }">
        <VAvatar>
          <VImg :src="item.profileImage" />
        </VAvatar>
      </template>

      <template #item.username="{ item }">
        <VBtn variant="text" @click="showUserDetails(item.username)">
          {{ item.username }}
        </VBtn>
      </template>

      <template #item.amount="{ item }">
        {{ item.amount.toLocaleString('tr-TR', { style: 'currency', currency: 'TRY' }) }}
      </template>
    </VDataTableServer>
  </div>
</template>
