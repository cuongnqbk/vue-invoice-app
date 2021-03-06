<template>
  <div class="container">
    
    <div class="c-overview">
      <VCard 
        v-for="(value, title, index) in overviewData"
        :key="title"
        :card-info="{ icon: icons[index], title: title, value }"
      />
    </div>
   
    <div class="main-cards">
      
      <div class="main-cards__container">
        <History :componentLoaded="componentLoaded" class="c-card c-hist" />
      </div>

      <div class="main-cards__container">
        <div class="c-card c-card--small-half c-vat">
          <div class="c-card__title">VAT</div>

          <div class="c-card__content">
            <div class="c-vat__item">
              <div class="c-vat__item--title">non-food VAT</div>
              
              <div class="c-vat__item--content">
                <input class="c-vat__input" type="text" placeholder="non-food VAT">
                <button class="c-vat__button" @click="addNewVat('non_food_vat', $event)">OK</button>
              </div>

              <div class="c-vat__current">Current: {{ vatData['non_food_vat'] || 'Not specified' }}</div>
            </div>

            <div class="c-vat__item">
              <div class="c-vat__item--title">food VAT</div>
                
                <div class="c-vat__item--content">
                  <input class="c-vat__input" type="text" placeholder="food VAT">
                  <button class="c-vat__button" @click="addNewVat('food_vat', $event)">OK</button>
                </div>
                
              <div class="c-vat__current">Current: {{ vatData['food_vat'] || 'Not specified' }}</div>
            </div>
          </div>
        </div>

        <div class="c-card c-card--big-half c-document">
          <div class="c-card__title">Recent Documents</div>
          
          <div class="c-card__content">
            <template v-if="shownDocuments.length">
              <div 
                class="c-document__item"
                v-for="(document, index) in shownDocuments"
                :key="document.id"
                @click="$router.push(`/documents/edit/${document.id}`)"
              >
                <div class="c-document__title">Document nr {{ documentsLength - index }}</div>
                <div class="c-document__provider">{{ document.provider_name }}</div>
                <div class="c-document__icon"><div class="icon-wrapper"><font-awesome-icon icon="file"/></div></div>
                <div class="c-document__date">{{ document.inserted_date }}</div>
              </div>
            </template>
            
            <template v-else-if="!shownDocuments.length && componentLoaded">
              <div class="h-centered">
                <p>No Documents</p>
              </div>
            </template>
          </div>
        </div>
      </div>
    </div>

    <HistoryModal />
  </div>
</template>

<script>
import VCard from '../components/VCard';
import VModal from '../components/VModal';
import History from '../components/dashboard/History';
import HistoryModal from '../components/dashboard/HistoryModal';

import { formatDate } from '../utils/';

const documentEntity = 'document';
const currentEntity = 'dashboard';

import * as common from '@/store/modules/common';
import { mapActions, mapState, mapGetters } from 'vuex';

import titleMixin from '../mixins/titleMixin';

export default {
  name: 'home',

  title: 'Dashboard',

  mixins: [titleMixin],

  components: { VCard, VModal, History, HistoryModal },

  data: () => ({
    icons: ['cart-plus', 'industry', 'file', 'clipboard-list'],
    shownDocumentsLen: 8,
    initialVat: {},
    componentLoaded: false,
    shouldUpdateDocument: false,
    vatHistoryMessage: null
  }),

  computed: {
    ...mapGetters(documentEntity, { documents: 'getItemsAsArr' }),

    ...mapState(currentEntity, {
      overviewData: 'dashboard/overview',
      vatData: 'vat',
      needsUpdate: 'needsUpdate',
      isInit: 'isInit',
    }),

    documentsLength () {
      return this.documents.length
    },

    shownDocuments () {
      return (this.documents.length > this.shownDocumentsLen 
        ? this.documents.slice(0, this.shownDocumentsLen)
        : this.documents).map(document => ({ ...document, inserted_date: formatDate(document.inserted_date) }))
    },
  },

  methods: {
    formatDate (date) { return formatDate(date) },

    ...mapActions(['changeEntity']),
    
    ...mapActions(currentEntity, ['fetchMainOverview', 'setNewVat', 'updateDocVat', 'insertHistoryRow', 'updateVat']),

    async addNewVat (type, ev) {
      const input = ev.target.previousElementSibling;
      const value = parseFloat(input.value.trim());

      if (isNaN(value) || value === this.vatData[type]) return;

      input.value = '';
      this.setNewVat({ type, value });

      await this.updateVat([type, value]);

      await this.updateDocVat([type, value]);

      // this.vatHistoryMessage = `Update ${type} from ${this.initialVat[type] === null ? "'Not Specified'" : this.initialVat[type]} to ${this.vatData[type]}`

      const message = `Update ${type} from ${this.initialVat[type] === null ? "'Not Specified'" : this.initialVat[type]} to ${this.vatData[type]}`;

      this.insertHistoryRow({ 
        entity: currentEntity, 
        message, 
        action_type: 'update' 
      });

      this.shouldUpdateDocument = true;
    },

    sendToRoute (newRoute) {
      return this.$router.push(newRoute)
    },
  },

  async created () {
    this.changeEntity('documents');
    !(this.$store && this.$store.state[documentEntity]) && (this.$store.registerModule(documentEntity, common))

    !(this.documents.length) && this.$store.dispatch('api/FETCH_DATA');

    // !this.historyData.length && this.fetchMainOverview();
    !this.isInit && await this.fetchMainOverview();

    this.initialVat = { ...this.vatData };

    this.componentLoaded = true;
  },

  async beforeRouteLeave (to, from, next) {
    
    if (this.shouldUpdateDocument) {
      // this.insertHistoryRow({ entity: currentEntity, message: this.vatHistoryMessage, action_type: 'update' });

      this.$store.dispatch('api/makeGETRequest', {
        url: this.$store.state['mainUrl'] + 'documents',
        entity: 'document'
      });
    }
    
    this.$children[1].closeModal();
    next();
  }
}
</script>


<style lang="scss" scoped>
  $row-title-length: 8rem;
  $main-blue: #394263;
  $main-blue-border: rgba($color: $main-blue, $alpha: .4);


  %title-style {
    color: #fff;
    background-color: lighten($color: $main-blue, $amount: 8%);
  }

  %icon-dark-bg {
    background-color: darken($color: $main-blue, $amount: 20%);
    border-radius: 50%;
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
    color: #DADAE3;
  }

  .h-centered, /deep/ .c-history .h-centered {
    width: 100%;
    height: 90%;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .container {
    width: 100%;
    height: 100%;
    padding: 2rem;
  }

  .c-overview {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(246px, 1fr));
    grid-gap: 30px;
    grid-auto-rows: 94px;
  }

  .main-cards {
    padding-right: 2rem;
    margin-top: 2.5rem;
    width: 100%;
    display: grid;
    grid-template-columns: 50% 50%;
    grid-column-gap: 30px;
    grid-template-rows: 600px;

    @media only screen and (max-width: 1150px) {
      &__container {
        grid-column: 1 / -1;
        &:last-child {
          margin-top: 2rem;
        }
      }
    }

    // .c-history - the History.vue' root class
    .c-card, /deep/ .c-history {
      background-color: #fff;
      display: flex;
      flex-direction: column;
      height: 100%;

      .c-card__title {
        @extend %title-style;
        flex-basis: 2rem;
        padding: 7px;
        text-align: center;
        letter-spacing: 1px;
        font-size: 1.3rem;
      }

      .c-card__content {
        flex: 1;
        height: 100%;
      }

      &--small-half {
        height: 120px;
      }

      &--big-half {
        margin-top: 50px;
        height: calc(100% - 170px);
      }
    }
  }

  .c-vat {
    
    .c-card__content {
      display: flex;
      justify-content: flex-start;
    }

    &__item {
      flex: 1;
      text-align: center;
      padding-top: 3px;

      &:first-child {
        border-right: 1px solid $main-blue-border;
      }

      &--title {
        color: darken($color: $main-blue, $amount: 30%);
        margin-bottom: 6px;
        border-bottom: 1px solid $main-blue-border;
      }

      &--content {
        padding-bottom: 4px;
        border-bottom: 1px solid $main-blue-border;

        .c-vat__input {
          margin-right: 1rem;
          padding: 3px;
          border-radius: 10px;
          border: 1px solid $main-blue-border;
          outline: none;
        }

        .c-vat__button {
          border-radius: 3px;
          color: #fff;
          background-color: $main-blue;
          border: none;
          padding: 3px;
          cursor: pointer;
        }
      }
    }

    &__current {
      @extend %title-style;
      letter-spacing: 1px;
      padding: 5px;
      font-size: 1.12rem;
    }
  }

  .c-document {
    overflow: auto;

    .c-card__content {
      padding: .8rem .9rem .3rem;
      display: flex;
      flex-wrap: wrap;
      justify-content: center;
    }

    &__item {
      background-color: $main-blue;
      color: #fff;
      flex: 0 0 130px;
      margin: 10px;
      transition: transform .3s;
      cursor: pointer;
      display: flex;
      flex-direction: column;
      justify-content: flex-start;
      align-items: center;
      min-height: 150px;
      max-height: 200px;

      &:hover {
        transform: translateY(-5px);
      }

    }

    .icon-wrapper {
      width: 3rem;
      height: 3rem;
      @extend %icon-dark-bg;
    }

    &__title {
      padding-top: 8px;
      font-style: italic;
      letter-spacing: calc(1.7 * 1px);
      font-size: .9rem;
    }

    &__provider {
      text-align: center;
      font-size: .85rem;
      font-weight: bold;
      margin-top: .7rem;
      padding: 0 .3rem;
      flex-grow: 0;
    }
    
    &__date {
      padding-bottom: 8px;
      font-weight: bold;
    }


    &__title, &__date {
      flex-basis: 10%;
    }

    &__icon {
      flex-basis: 80%;
      display: flex;
      align-items: center;
    }
  }
</style>