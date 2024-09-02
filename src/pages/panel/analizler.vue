<script setup>
import { debounce } from 'lodash';
import { onMounted, ref, watch } from 'vue';

// Stateler
const items = ref([]);
const totalItems = ref(0);
const itemsPerPage = ref(10);
const currentPage = ref(1);
const searchTerms = ref({
    username: '',
    league: '',
    team1: '',
    team2: '',
    comment: '',
    odd_val: '',
    status: '',
    view: '',
    created_date: '',
    full_name: ''
});
const headers = [
    { title: "Kullanıcı", key: "username" },
    { title: "Ad Soyad", key: "full_name" },
    { title: "Lig", key: "league" },
    { title: "Takımlar", key: "teams" },
    { title: "Oyun", key: "game" },
    { title: "Yorum", key: "comment" },
    { title: "Oran", key: "odd_val" },
    { title: "Durum", key: "status" },
    { title: "Görüntülenme", key: "view" },
    { title: "Oluşturulma Tarihi", key: "created_date" },
];

const statusOptions = [
    { text: 'Onaylandı', value: '1' },
    { text: 'Bekliyor', value: '0' }
];

const bearerToken = '23bb3f1da1b689d76800bd3f54f373b9fabe67adb72dcedd7895c46ecce7ca2f';
const apiUrl = 'http://127.0.0.1:8011/api/system/v1/comments/';

const fetchData = async ({ page, itemsPerPage, searchTerms }) => {
    try {
        const skip = (page - 1) * itemsPerPage;
        const take = itemsPerPage;
        const filters = Object.entries(searchTerms)
            .filter(([, value]) => value)
            .map(([key, value]) => {
                if (key === 'username') {
                    return `["user.username","contains","${value}"]`;
                }
                if (key === 'full_name') {
                    return `["user.first_name","contains","${value}"],["user.last_name","contains","${value}"]`;
                }
                if (key === 'league') {
                    return `["odd.game.bet.league.name","contains","${value}"]`;
                }
                if (key === 'team1') {
                    return `["odd.game.bet.team1.name","contains","${value}"]`;
                }
                if (key === 'team2') {
                    return `["odd.game.bet.team2.name","contains","${value}"]`;
                }
                if (key === 'comment') {
                    return `["comment","contains","${value}"]`;
                }
                if (key === 'odd_val') {
                    return `["odd_val","contains","${value}"]`;
                }
                if (key === 'status') {
                    return `["status","=","${value}"]`;
                }
                if (key === 'view') {
                    return `["view","contains","${value}"]`;
                }
                if (key === 'created_date') {
                        const date = new Date(value);
                        date.setDate(date.getDate() - 1);
                        value = date.toISOString().split('T')[0];
                    return `["created_date","contains","${value}"]`;
                }
            });

        const filterQuery = filters.length > 0 ? `filter=[${filters.join(',"and",')}]` : '';

        const response = await fetch(`${apiUrl}?take=${take}&skip=${skip}&${filterQuery}`, {
            method: 'GET',
            headers: {
                'Authorization': `Token ${bearerToken}`,
                'Content-Type': 'application/json',
            },
        });

        if (response.ok) {
            const data = await response.json();
            items.value = data.data.map(comment => ({
                id: comment.id,
                username: comment.user.username,
                full_name: `${comment.user.first_name} ${comment.user.last_name}`,
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
            }));
            totalItems.value = data.totalCount;
        } else {
            console.error('Failed to fetch data:', response.statusText);
        }
    } catch (error) {
        console.error('Error:', error);
    }
};

const debouncedFetchData = debounce(fetchData, 300);

watch([currentPage, itemsPerPage], () => {
    debouncedFetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value });
});

watch(searchTerms, () => {
    currentPage.value = 1;
    debouncedFetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value });
}, { deep: true });

onMounted(() => {
    fetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value });
});

const formatDate = (date) => {
    if (!date) return '';
    const d = new Date(date);
    return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`;
};
</script>

<template>
    <div>
        <VCard class="mb-6" title="Tüm Analizler">
            <VCardText>Kullanıcılar tarafından yapılan analizlerini ve durumlarını gösterir.</VCardText>
        </VCard>

        <VCard class="mb-6">
            <VCardText>
                <VRow>
                    <VCol v-for="header in headers" :key="header.key" cols="12" sm="6" md="4" lg="3">
                        <VTextField
                            v-if="header.key !== 'teams' && header.key !== 'status' && header.key !== 'created_date'"
                            v-model="searchTerms[header.key]"
                            :label="header.title"
                            dense
                            hide-details
                            outlined
                        />
                        <VSelect
                            v-else-if="header.key === 'status'"
                            v-model="searchTerms.status"
                            :items="statusOptions"
                            item-title="text"
                            item-value="value"
                            label="Durum"
                            placeholder="Durum seçiniz..."
                            dense
                            hide-details
                            outlined
                        />
                        <AppDateTimePicker
                            v-else-if="header.key === 'created_date'"
                            v-model="searchTerms.created_date"
                            placeholder="Oluşturulma Tarihi"
                            @update:modelValue="searchTerms.created_date = formatDate($event)"
                        />
                        <VRow v-else-if="header.key === 'teams'">
                            <VCol cols="6">
                                <VTextField
                                    v-model="searchTerms.team1"
                                    label="Takım 1"
                                    dense
                                    hide-details
                                    outlined
                                />
                            </VCol>
                            <VCol cols="6">
                                <VTextField
                                    v-model="searchTerms.team2"
                                    label="Takım 2"
                                    dense
                                    hide-details
                                    outlined
                                />
                            </VCol>
                        </VRow>
                    </VCol>
                </VRow>
            </VCardText>
        </VCard>

        <VDataTableServer
            :items="items"
            :headers="headers"
            :items-length="totalItems"
            :items-per-page="itemsPerPage"
            v-model:page="currentPage"
            @update:options="fetchData"
        >
            <!-- Status Column Slot -->
            <template v-slot:item.status="{ item }">
                <VChip :color="item.status === 1 ? 'success' : 'warning'">
                    {{ item.status === 1 ? 'Onaylandı' : 'Bekliyor' }}
                </VChip>
            </template>
        </VDataTableServer>
    </div>
</template>
