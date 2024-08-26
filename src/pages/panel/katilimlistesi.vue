<script setup>
import { onMounted, ref, watch } from 'vue';

// Stateler
const items = ref([]);
const totalItems = ref(0);
const itemsPerPage = ref(10);
const currentPage = ref(1);
const searchTerms = ref({
    username: '',
    league: '',
    teams: '',
    game: '',
    comment: '',
    odd_val: '',
    status: '',
    view: '',
    created_date: ''
});
const headers = [
    { title: "Kullanıcı", key: "username" },
    { title: "Seviye", key: "experience" },
    { title: "Boost Sayısı", key: "type" },
    { title: "Oyun", key: "game" },
    { title: "Yorum", key: "comment" },
    { title: "Oran", key: "odd_val" },
    { title: "Durum", key: "status" },
    { title: "Oluşturulma Tarihi", key: "created_date" },
];

const bearerToken = 'c11ca19b9a2de7f7990d55ef8a8c765967e498e45cd45a879b1e93d7becb9493';
const apiUrl = 'https://server.motiwy.com/api/system/v1/comments/';

const fetchData = async ({ page, itemsPerPage, searchTerms }) => {
    try {
        const skip = (page - 1) * itemsPerPage;
        const take = itemsPerPage;
        const filters = Object.entries(searchTerms)
            .filter(([, value]) => value)
            .map(([key, value]) => `["${key}","contains","${value}"]`);

        const filterQuery = filters.length > 0 ? `filter=[${filters.join(',')}]` : '';

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

watch([currentPage, itemsPerPage, searchTerms], () => {
    fetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value });
});

onMounted(() => {
    fetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value });
});
</script>

<template>
    <div>
        <VCard class="mb-6" title="Katılım Listesi">
            <VCardText>Boost katılımı sağlayan kullanıcıların listesini gösterir.</VCardText>
        </VCard>

        <v-data-table-server :items="items" :headers="headers" :items-length="totalItems" :items-per-page="itemsPerPage"
            :page="currentPage" @update:options="fetchData">



            <!-- Headers with Search Inputs -->
            <template v-slot:header="headerProps">
                <tr>
                    <th v-for="header in headerProps.headers" :key="header.key">
                        <VTextField v-model="searchTerms[header.key]" :label="header.title" dense hide-details solo
                            @input="fetchData({ page: currentPage.value, itemsPerPage: itemsPerPage.value, searchTerms: searchTerms.value })" />
                    </th>
                </tr>
            </template>

            <!-- Status Column Slot -->
            <template v-slot:item.status="{ item }">
                <template v-if="item.status === 1">
                    <VChip color="success">
                        Onaylandı
                    </VChip>
                </template>
                <template v-else>
                    Pasif
                </template>
            </template>

        </v-data-table-server>
    </div>
</template>
