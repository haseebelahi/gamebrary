<template lang="html">
    <div
        class="lists"
        ref="lists"
        v-if="user && platform"
        :class="{ dark: darkModeEnabled }"
        @click.self="loseFocus"
    >
        <game-board-placeholder :id="gameDetailId" v-if="loading" />

        <modal
            ref="game"
            large
            no-padding
            @close="closeGame"
        >
            <game-detail
                v-if="gameDetailId"
                slot="content"
                :id="gameDetailId"
                :list-id="gameDetailListIndex"
            />
        </modal>

        <modal
            ref="tag"
            :title="$t('tags.applyTag')"
            :message="$t('tags.useTags')"
            padded
        >
            <div slot="content">
                <div
                    class="tags"
                    v-for="(tag, name) in tags"
                    :key="name"
                >
                    <tag
                        :label="name"
                        :hex="tag.hex"
                        :readonly="!tag.games.includes(gameTagsId)"
                        @action="tryAdd(tag.games, name)"
                        @close="removeTag(name)"
                    />
                </div>
            </div>
        </modal>

        <template>
            <list
                :name="list.name"
                :games="list.games"
                :listIndex="listIndex"
                :key="`${list.name}-${listIndex}`"
                v-if="list && !loading"
                v-for="(list, listIndex) in gameLists[platform.code]"
                @end="dragEnd"
            />

            <list-add
                v-if="addingList || !list"
                @scroll="scroll"
                @update="updateLists"
            />

            <game-board-actions
                v-else
                @update="updateLists"
                @scroll="scroll"
            />
        </template>

        <!-- <dev-debug /> -->
    </div>
</template>

<script>
import GameBoardActions from '@/components/GameBoard/GameBoardActions';
import GameBoardPlaceholder from '@/components/GameBoard/GameBoardPlaceholder';
import Tag from '@/components/Tags/Tag';
import ListAdd from '@/components/Lists/ListAdd';
import Panel from '@/components/Panel/Panel';
import GameDetail from '@/pages/GameDetail';
import Modal from '@/components/Modal/Modal';
// import DevDebug from '@/components/DevDebug/DevDebug';
import List from '@/components/Lists/List';
import draggable from 'vuedraggable';
import { mapState, mapGetters } from 'vuex';
import firebase from 'firebase/app';
import 'firebase/firestore';

const db = firebase.firestore();

export default {
    components: {
        draggable,
        List,
        // DevDebug,
        GameBoardActions,
        GameBoardPlaceholder,
        ListAdd,
        Tag,
        Panel,
        GameDetail,
        Modal,
    },

    data() {
        return {
            dragging: false,
            draggingId: null,
            loading: false,
            gameData: null,
            gameDetailListIndex: null,
            gameDetailId: null,
            gameTagsId: null,
        };
    },

    computed: {
        ...mapState(['user', 'gameLists', 'addingList', 'platform', 'tags', 'games']),
        ...mapGetters(['darkModeEnabled']),

        list() {
            return this.gameLists && this.platform && this.gameLists[this.platform.code]
                ? this.gameLists[this.platform.code]
                : null;
        },
    },

    mounted() {
        this.$store.commit('CLEAR_ACTIVE_LIST_INDEX');

        if (this.platform || this.$route.name === 'share-list') {
            this.loadGameData();
            this.setPageTitle();
        } else {
            this.$router.push({ name: 'platforms' });
        }

        this.$bus.$on('OPEN_GAME', this.openGame);
        this.$bus.$on('OPEN_TAGS', this.openTags);
    },

    beforeDestroy() {
        this.$bus.$off('OPEN_GAME');
        this.$bus.$off('OPEN_TAGS');
    },

    methods: {
        tryAdd(games, tagName) {
            if (!games.includes(this.gameTagsId)) {
                this.addTag(tagName);
            }
        },

        addTag(tagName) {
            this.$store.commit('ADD_GAME_TAG', { tagName, gameId: this.gameTagsId });
            this.$bus.$emit('SAVE_TAGS', this.tags);
        },

        closeGame() {
            this.setPageTitle();
            this.gameDetailId = null;
        },

        setPageTitle() {
            document.title = this.platform
                ? `${this.platform.name} - Gamebrary`
                : 'Gamebrary';
        },

        removeTag(tagName) {
            this.$store.commit('REMOVE_GAME_TAG', { tagName, gameId: this.gameTagsId });
            this.$bus.$emit('SAVE_TAGS', this.tags);
        },

        openGame({ id, listId }) {
            this.gameDetailId = id;
            this.gameDetailListIndex = listId;
            this.$refs.game.open();
        },

        openTags(id) {
            this.gameTagsId = id;
            this.$refs.tag.open(id);
        },

        loseFocus() {
            this.$store.commit('CLEAR_ACTIVE_LIST_INDEX');
        },

        scroll() {
            this.$nextTick(() => {
                const lists = this.$refs.lists;
                lists.scrollLeft = lists.scrollWidth;
            });
        },

        dragEnd() {
            this.dragging = false;
            this.draggingId = null;
            this.$bus.$emit('TOAST', { message: 'Collection updated' });
            this.updateLists();
        },

        updateLists(force) {
            db.collection('lists').doc(this.user.uid).set(this.gameLists, { merge: !force })
                .catch(() => {
                    this.$bus.$emit('TOAST', { message: 'Authentication error', type: 'error' });
                });
        },

        loadGameData() {
            if (this.list) {
                const gameList = [];

                this.list.forEach((list) => {
                    if (list && list.games.length) {
                        list.games.forEach((id) => {
                            if (!gameList.includes(id)) {
                                gameList.push(id);
                            }
                        });
                    }
                });

                if (gameList.length > 0) {
                    this.loading = true;

                    this.$store.dispatch('LOAD_GAMES', gameList)
                        .then(() => {
                            this.loading = false;
                        })
                        .catch(() => {
                            this.$bus.$emit('TOAST', { message: 'Error loading game', type: 'error' });
                        });
                }
            }
        },
    },
};
</script>

<style lang="scss" rel="stylesheet/scss" scoped>
    @import "~styles/styles.scss";

    .draggable * {
        color: $color-white;
    }

    .panel {
        margin-right: $gp;
        width: $list-width;
    }

    .lists {
        user-select: none;
        display: flex;
        align-items: flex-start;
        height: calc(100vh - 48px);
        padding: 0 $gp;
        box-sizing: border-box;
        overflow-x: auto;
        overflow-x: overlay;
        display: flex;

        &.empty {
            background: $color-white;
        }
    }

    .list-placeholder {
        opacity: 0.25;
    }
</style>
