<template lang="html">
    <div :class="['platforms-page', { dark: darkModeEnabled }]">

        <div class="platforms">
            <platform
                v-for="platform in filteredPlatforms"
                :key="platform.name"
                :platform-data="platform"
                :block-height="platform.height"
                :block-width="platform.width"
            />
        </div>

        <footer>
            <small>
              {{ $t('platforms.donateMessage') }}
                <a href="https://www.paypal.me/RomanCervantes/5" target="_blank">
                    {{ $t('platforms.donating') }}
                </a>
                ,
                <a href="https://github.com/romancmx/gamebrary/issues" target="_blank">
                    {{ $t('platforms.reportBugs') }}
                </a>
                {{$t('global.or')}}
                <a href="https://goo.gl/forms/r0juBCsZaUtJ03qb2" target="_blank">
                    {{ $t('platforms.submitFeedback') }}
                </a>
                .
            </small>

            <igdb-credit gray />
        </footer>
    </div>
</template>

<script>
import Masonry from 'masonry-layout';
import platforms from '@/shared/platforms';
import ToggleSwitch from '@/components/ToggleSwitch/ToggleSwitch';
import IgdbCredit from '@/components/IgdbCredit/IgdbCredit';
import Platform from '@/components/Platform/Platform';
import Panel from '@/components/Panel/Panel';
import { sortBy } from 'lodash';
import { mapState, mapGetters } from 'vuex';

let msnry = null;

export default {
    components: {
        ToggleSwitch,
        IgdbCredit,
        Platform,
        Panel,
    },

    data() {
        return {
            platforms,
        };
    },

    computed: {
        ...mapState(['gameLists', 'platform', 'settings']),
        ...mapGetters(['darkModeEnabled']),

        hasLists() {
            return Object.keys(this.gameLists).length > 0;
        },

        ownedListsOnly() {
            return this.settings && this.settings.ownedListsOnly;
        },

        filteredPlatforms() {
            const availableLists = this.ownedListsOnly
                ? this.platforms.filter(({ code }) => this.gameLists[code])
                : this.platforms;

            if (msnry) {
                msnry.reloadItems();
                msnry.layout();
            }

            return this.settings && this.settings.sortListsAlphabetically
                ? sortBy(availableLists, 'name')
                : sortBy(availableLists, 'generation').reverse();
        },
    },

    mounted() {
        this.initGrid();
    },

    methods: {
        initGrid() {
            msnry = new Masonry('.platforms', {
                itemSelector: '.platform',
                gutter: 10,
            });
        },

        groupLabel(label) {
            return label === '0'
                ? this.$t('platforms.computersArcade')
                : `${this.ordinalSuffix(label)} ${this.$t('platforms.generation')}`;
        },

        ordinalSuffix(value) {
            const j = value % 10;
            const k = value % 100;

            if (j === 1 && k !== 11) {
                return `${value}${this.$t('platforms.st')}`;
            }

            if (j === 2 && k !== 12) {
                return `${value}${this.$t('platforms.nd')}`;
            }
            if (j === 3 && k !== 13) {
                return `${value}${this.$t('platforms.rd')}`;
            }

            return `${value}${this.$t('platforms.th')}`;
        },
    },
};
</script>

<style lang="scss" rel="stylesheet/scss" scoped>
    @import "~styles/styles.scss";

    .platforms-page {
        color: $color-dark-gray;
        min-height: calc(100vh - #{$navHeight});
        padding: 0 $gp;

        &.dark {
            color: $color-gray;

            .group-label {
                background-color: $color-darkest-gray;
            }
        }
    }

    .platforms {
        display: flex;
        flex-direction: column;

        &.reverse {
            flex-direction: column-reverse;
        }
    }

    .group-label {
        position: sticky;
        top: 0;
        padding: $gp / 3 0;
        background-color: $color-gray;
    }

    footer {
        padding: $gp / 2 0;
        justify-content: center;
        display: flex;
        align-items: center;

        a {
            color: $color-dark-gray;
        }
    }
</style>
