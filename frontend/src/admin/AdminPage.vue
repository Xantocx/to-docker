<template>
    <b-container fluid>
        <b-row>
            <b-col cols="9">
                <div v-if="page === 'users'">
                    <users-page></users-page>
                </div>
            </b-col>
        </b-row>
    </b-container>
</template>

<script>
import {get} from '@/utils/request'
import {getCookie} from '@/utils/cookies'

import UsersPage from './components/UsersPage.vue'

export default {
  components: { UsersPage },
    name: 'admin-page',
    created () {
        let jwt_token = getCookie("jwt")
        if (!jwt_token) {
            this.$router.push({'name': 'login'})
            return
        }
        get(`check-login`, {'headers': {'Authorization': `bearer ${jwt_token}`}})
        .then((response) => {
            if (response.status === 401) {
                this.$router.push({'name': 'login'})
            }
        })
    },
    data () {
        return {
            page: 'users'
        }
    },
    methods: {
        setPage (page) {
            this.page = page;
        }
    }
}
</script>

<style scoped>
.menu-item {
    margin-bottom: 2rem;
    cursor:pointer;
    text-align: left;
}
</style>