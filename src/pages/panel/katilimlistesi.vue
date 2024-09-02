<script setup>
import { onMounted, ref, watch } from 'vue';
import { useRouter } from 'vue-router';

const router = useRouter();

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

    filters.push(`["type","=","1"]`)

    const filterQuery = `filter=[${filters.join(',')}]`

    const url = `http://127.0.0.1:8011/api/system/v1/comments/?take=${take}&skip=${skip}&${filterQuery}`
    const bearerToken = '23bb3f1da1b689d76800bd3f54f373b9fabe67adb72dcedd7895c46ecce7ca2f';

    const response = await fetch(url, {
      method: 'GET',
      headers: {
        'Authorization': `Token ${bearerToken}`,
        'Content-Type': 'application/json',
      },
    });

    if (!response.ok) {
      throw new Error(`HTTP hatası! durum: ${response.status}`);
    }

    const { data, totalCount } = await response.json();

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
    console.error('Hata:', error)
  }
}

watch([currentPage, itemsPerPage, searchTerms], () => {
  fetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value })
})

onMounted(() => {
  fetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value })
})

const editItem = (item) => {
  console.log('Öğeyi düzenle:', item)
}

const deleteItem = (item) => {
  console.log('Öğeyi sil:', item)
}

const showUserDetails = (username) => {
  router.push({ name: 'user-details', params: { username } });
}

const isModalVisible = ref(false)
const selectedUser = ref(null)
const isLoading = ref(false)

const showModal = async (user) => {
  isLoading.value = true;
  isModalVisible.value = true;
  try {
    const userDetails = await fetchUserDetails(user.username);
    selectedUser.value = {
      ...user,
      ...userDetails,
    };
  } catch (error) {
    console.error('Kullanıcı detayları alınırken hata oluştu:', error);
  } finally {
    isLoading.value = false;
  }
}

const fetchUserDetails = async (username) => {
  try {
    const url = `http://127.0.0.1:8011/api/system/v1/users/?filter=["username","=","${username}"]`;
    const bearerToken = '23bb3f1da1b689d76800bd3f54f373b9fabe67adb72dcedd7895c46ecce7ca2f';

    const response = await fetch(url, {
      method: 'GET',
      headers: {
        'Authorization': `Token ${bearerToken}`,
        'Content-Type': 'application/json',
      },
    });

    if (!response.ok) {
      throw new Error(`HTTP hatası! durum: ${response.status}`);
    }

    const { data, totalCount } = await response.json();

    if (totalCount === 0 || !data[0]) {
      throw new Error('Kullanıcı bulunamadı');
    }

    const user = data[0];

    return {
      id: user.id,
      username: user.username,
      firstName: user.first_name,
      lastName: user.last_name,
      avatar: user.avatar,
      experience: user.experience,
      profileView: user.profile_view,
      commentator: user.commentator,
      commentatorEndDate: user.commentator_end_date,
      highlightEndDate: user.highlight_end_date,
      blueTick: user.blue_tick,
      boostCount: user.boost_count,
      totalEarnedPoints: user.total_earned_points,
      totalSpentPoints: user.total_spent_points,
      transactionHistory: user.transaction_history,
    };
  } catch (error) {
    console.error('Kullanıcı detayları alınırken hata oluştu:', error);
    throw error;
  }
}

const analysisData = ref({
  winners: 0,
  losers: 0,
  total: 0
})

const dailyDistribution = ref({
  date: (() => {
    const date = new Date();
    date.setDate(date.getDate() - 1);
    return date.toLocaleDateString('tr-TR');
  })(),
  winnerCount: 0,
  winRate: 0,
  totalPointsDistributed: 0,
  remainingPoints: 0
})

const fetchAnalysisData = async () => {
  try {
    const response = await fetch('http://127.0.0.1:8011/api/system/v1/analysis')
    const data = await response.json()
    analysisData.value = data
  } catch (error) {
    console.error('Analiz verileri alınırken hata oluştu:', error)
  }
}

const fetchDailyDistribution = async () => {
  try {
    const response = await fetch('http://127.0.0.1:8011/api/system/v1/daily-distribution')
    const data = await response.json()
    dailyDistribution.value = data
  } catch (error) {
    console.error('Günlük dağılım verileri alınırken hata oluştu:', error)
  }
}

onMounted(() => {
  fetchAnalysisData()
  fetchDailyDistribution()
  fetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value })
})
</script>

<template>
  <div>
    <VCard class="mb-6" title="Katılım Listesi">
      <VCardText>Boost katılımı sağlayan kullanıcıların listesini gösterir.</VCardText>
    </VCard>

    <VRow class="mb-6">
      <VCol cols="12" md="4">
        <VCard>
          <VCardTitle>Kazanan</VCardTitle>
          <VCardText class="text-h4 text-center">{{ analysisData.winners }}</VCardText>
        </VCard>
      </VCol>
      <VCol cols="12" md="4">
        <VCard>
          <VCardTitle>Kaybeden</VCardTitle>
          <VCardText class="text-h4 text-center">{{ analysisData.losers }}</VCardText>
        </VCard>
      </VCol>
      <VCol cols="12" md="4">
        <VCard>
          <VCardTitle>Toplam</VCardTitle>
          <VCardText class="text-h4 text-center">{{ analysisData.total }}</VCardText>
        </VCard>
      </VCol>
    </VRow>

    <VCard class="mb-6">
      <VCardTitle>Günün Kazananı ({{ dailyDistribution.date }})</VCardTitle>
      <VCardText>
        <VRow>
          <VCol cols="12" sm="6" md="3">
            <strong>Kazanan Sayısı:</strong> {{ dailyDistribution.winnerCount }}
          </VCol>
          <VCol cols="12" sm="6" md="3">
            <strong>Kazanç Oranı:</strong> {{ dailyDistribution.winRate }}%
          </VCol>
          <VCol cols="12" sm="6" md="3">
            <strong>Dağıtılan Toplam Puan:</strong> {{ dailyDistribution.totalPointsDistributed }}
          </VCol>
          <VCol cols="12" sm="6" md="3">
            <strong>Kasada Kalan Puan:</strong> {{ dailyDistribution.remainingPoints }}
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
      hover
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

      <template #item="{ item }">
        <tr 
          @click="showModal(item)" 
          style="cursor: pointer;"
          class="table-row-hover"
        >
          <td>{{ item.username }}</td>
          <td>{{ item.experience }}</td>
          <td>{{ item.type }}</td>
          <td>{{ item.game }}</td>
          <td>{{ item.comment }}</td>
          <td>{{ item.odd_val }}</td>
          <td>
            <VChip :color="item.status === 1 ? 'success' : 'error'">
              {{ item.status === 1 ? 'Onaylandı' : 'Pasif' }}
            </VChip>
          </td>
          <td>{{ item.created_date }}</td>
          <td>
            <VBtn icon="tabler-pencil" size="small" @click.stop="editItem(item)" class="mr-2" />
            <VBtn icon="tabler-file-x" size="small" @click.stop="deleteItem(item)" />
          </td>
        </tr>
      </template>
    </VDataTableServer>

    <VDialog v-model="isModalVisible" max-width="800px">
      <VCard>
        <VCardTitle>
          <span class="text-h5">Kullanıcı Detayları</span>
        </VCardTitle>
        <VCardText>
          <VRow v-if="isLoading" justify="center" align="center" style="height: 200px;">
            <VCol cols="auto">
              <VProgressCircular
                indeterminate
                color="primary"
                size="64"
              />
            </VCol>
          </VRow>
          <div v-else-if="selectedUser">
            <VRow>
              <VCol cols="12" sm="4">
                <VAvatar size="120">
                  <VImg :src="selectedUser.avatar" />
                </VAvatar>
              </VCol>
              <VCol cols="12" sm="8">
                <p><strong>Kullanıcı Adı:</strong> {{ selectedUser.username }}</p>
                <p><strong>Ad Soyad:</strong> {{ selectedUser.firstName }} {{ selectedUser.lastName }}</p>
                <p><strong>Seviye:</strong> {{ selectedUser.experience }}</p>
                <p><strong>Profil Görüntülenme:</strong> {{ selectedUser.profileView }}</p>
                <p><strong>Yorumcu Seviyesi:</strong> {{ selectedUser.commentator }}</p>
                <p><strong>Yorumcu Bitiş Tarihi:</strong> {{ new Date(selectedUser.commentatorEndDate).toLocaleString('tr-TR') }}</p>
                <p><strong>Öne Çıkarma Bitiş Tarihi:</strong> {{ new Date(selectedUser.highlightEndDate).toLocaleString('tr-TR') }}</p>
                <VChip :color="selectedUser.blueTick ? 'primary' : 'grey'" class="mt-2">
                  {{ selectedUser.blueTick ? 'Mavi Tik' : 'Mavi Tik Yok' }}
                </VChip>
              </VCol>
            </VRow>
            <VDivider class="my-4"></VDivider>
            <h3 class="mb-4">İşlem Geçmişi</h3>
            <VDataTable
              :headers="[
                { title: 'Tarih', key: 'date' },
                { title: 'İşlem', key: 'transaction' },
                { title: 'Miktar', key: 'amount' },
              ]"
              :items="selectedUser.transactionHistory || []"
            ></VDataTable>
          </div>
          <div v-else>
            <p>Kullanıcı bilgileri yüklenemedi.</p>
          </div>
        </VCardText>
        <VCardActions>
          <VSpacer></VSpacer>
          <VBtn color="blue darken-1" text @click="isModalVisible = false">Kapat</VBtn>
        </VCardActions>
      </VCard>
    </VDialog>
  </div>
</template>

<style scoped>
.table-row-hover:hover {
  background-color: rgba(0, 0, 0, 0.1);
  transition: background-color 0.3s ease;
}
</style>
