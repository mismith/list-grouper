<template>
  <div id="app">
    <md-toolbar class="md-dense">
      <div>
        <h1 class="md-title" style="margin-right: auto;">List Grouper</h1>

        <!--<md-button @click.native="createList()" class="md-icon-button">
          <md-icon>add_circle_outline</md-icon>
          <md-tooltip>New list</md-tooltip>
        </md-button>-->

        <md-button v-if="!loading.upload" class="md-icon-button file-upload-button">
          <md-icon>file_upload</md-icon>
          <input type="file" accept="text/plain,application/json" @change="handleFileUpload($event.target.files[0])" @drop="handleFileUpload($event.dataTransfer.files[0])" />
          <md-tooltip>Upload</md-tooltip>
        </md-button>
        <md-spinner v-else />
      </div>
      <div>
        <md-input-container md-inline>
          <md-input v-model="list.name" @change="dirty = true;" placeholder="Untitled list" />
        </md-input-container>

        <md-button v-if="!loading.save" @click.native="save()" class="md-icon-button" :class="{'md-raised md-accent': dirty}">
          <md-icon>call_merge</md-icon>
          <md-tooltip>Save</md-tooltip>
        </md-button>
        <md-spinner v-else />

        <md-button v-if="!loading.duplicate" @click.native="save(true)" class="md-icon-button">
          <md-icon>call_split</md-icon>
          <md-tooltip>Duplicate</md-tooltip>
        </md-button>
        <md-spinner v-else />

        <md-menu v-if="!loading.download" md-size="3">
          <md-button class="md-icon-button" md-menu-trigger>
            <md-icon>file_download</md-icon>
            <md-tooltip>Download</md-tooltip>
          </md-button>
          <md-menu-content>
            <md-menu-item @click.native="exportJson(uploadedFilename)">
              <md-icon>file_download</md-icon>
              <span>JSON</span>
            </md-menu-item>
            <md-menu-item @click.native="exportCsv(uploadedFilename)">
              <md-icon>file_download</md-icon>
              <span>CSV</span>
            </md-menu-item>
          </md-menu-content>
        </md-menu>
        <md-spinner v-else />

        <span style="flex-grow: 1;"></span>

        <div v-if="meLoaded">
          <md-button v-if="!me" @click.native="login()" class="md-raised">
            <img src="/static/google-logo.svg" alt="Google" width="20" style="margin-top: -1px; margin-left: -5px; margin-right: 5px;" /> Login
          </md-button>
          <div v-else>
            <md-menu md-size="3">
              <md-button md-menu-trigger class="md-icon-button" style="padding: 0;">
                <md-avatar>
                  <img :src="me.photoURL" role="presentation" />
                  <md-tooltip>Account</md-tooltip>
                </md-avatar>
              </md-button>
              <md-menu-content>
                <md-menu-item @click.native="logout()">
                  <md-icon>power_settings_new</md-icon>
                  <span>Logout</span>
                </md-menu-item>
              </md-menu-content>
            </md-menu>
          </div>
        </div>
        <md-spinner v-else />
      </div>
    </md-toolbar>

    <data-grouper
      v-if="list.data"
      ref="dataGrouper"
      :initial-list="list"
      @dirty="dirty = true;"
    />
    <aside v-else>
      <div>No data loaded.</div>
    </aside>

    <md-dialog-confirm
      ref="loginConfirm"
      md-content-html="You need to login first."
      md-ok-text="Login"
      md-cancel-text="Cancel"
    />
    <md-snackbar ref="snackbar" md-position="bottom right" @click.native="$refs.snackbar.close()">
      Saved!
    </md-snackbar>
  </div>
</template>

<script>
// import moment from 'moment-mini';
import CSVStringify from 'csv-stringify';
import FirebaseAuthMixin from '@/mixins/FirebaseAuth';
import {
  idKey,
  db,
} from '@/helpers/firebase';
import DataGrouper from './components/DataGrouper';

export default {
  name: 'app',
  mixins: [
    FirebaseAuthMixin,
  ],
  data() {
    return {
      file: undefined,
      list: {
        name: undefined,
      },
      listId: undefined,
      dirty: false,
      loading: {
        upload: false,
        download: false,
        save: false,
        duplicate: false,
      },
    };
  },
  computed: {
    uploadedFilename() {
      return (this.file && this.file.name && this.file.name.replace(/\.\w+$/, '')) || undefined;
    },
  },
  methods: {
    // import
    handleFileUpload(file) {
      this.loading.upload = true;
      this.list.data = undefined;
      this.file = file;

      // since this seems to lock the browser, make sure the loading icon is showing first
      this.$nextTick(() => {
        const reader = new FileReader();
        reader.onload = (e) => {
          const content = e.target.result;
          switch (file.type) {
            case 'application/json':
              this.parseJson(JSON.parse(content));
              break;
            case 'text/plain':
            default:
              this.parseText(content);
              break;
          }
          this.loading.upload = false;
        };
        reader.readAsText(file);
      });
    },

    fetchText(url) {
      return fetch(url)
        .then(res => res.text());
    },
    fetchJson(url) {
      return fetch(url)
        .then(res => res.json());
    },

    parseText(text) {
      const data = [];
      const lines = text.replace(/^\s+|\s+$/ig, '').split('\n');
      lines.forEach((line, i) => {
        const value = line.replace(/^\s+|\s+$/ig, '');
        if (value !== '') { // skip empty lines
          data.push({
            key: i,
            value,
            votes: 1,
          });
        }
      });
      this.list.data = data;
      this.list.name = this.uploadedFilename;
    },
    parseJson(json) {
      if (json.data) {
        this.list = json;
        this.list.name = this.list.name || this.uploadedFilename;
      } else {
        this.list.data = json;
        this.list.name = this.uploadedFilename;
      }
    },

    loadText(url) {
      return this.fetchText(url)
        .then(this.parseText);
    },
    loadJson(url) {
      return this.fetchJson(url)
        .then(this.parseJson);
    },

    // export
    download(contents, filename = 'grouped', extension = 'txt') {
      const a = document.createElement('a');
      let mime;
      switch (extension) {
        case 'json':
          mime = 'application/json';
          break;
        case 'csv':
          mime = 'text/csv';
          break;
        case 'txt':
        default:
          mime = 'text/plain';
          break;
      }
      a.setAttribute('href', `data:${mime};charset=utf-8,${encodeURIComponent(contents)}`);
      a.setAttribute('download', `${filename}.${extension}`);
      a.click();
    },
    exportJson(filename = undefined) {
      this.loading.download = true;
      const jsonBundle = this.$refs.dataGrouper.bundleJson();

      const json = JSON.stringify(jsonBundle, null, 2);

      this.download(json, filename, 'json');
      this.loading.download = false;
    },
    exportCsv(filename = undefined) {
      this.loading.download = true;
      const jsonBundle = this.$refs.dataGrouper.bundleJson();

      const rows = jsonBundle.data.map(datum => ({
        ...datum,
        // turn the group number into the group name
        group: jsonBundle.groupNames[datum.group],
      }));
      CSVStringify(rows, (err, csv) => {
        if (err) throw new Error(err);

        this.download(csv, filename, 'csv');
        this.loading.download = false;
      });
    },

    // CRUD
    loginIfNecessary() {
      if (!this.me) {
        return new Promise((resolve, reject) => {
          this.$refs.loginConfirm.open();
          this.$refs.loginConfirm.$on('close', (state) => {
            if (state === 'ok') return resolve();
            return reject('User cancelled');
          });
        })
          .then(this.login)
          .then(() => new Promise((resolve) => {
            // ensure `me` is ready before proceeding
            const unwatch = this.$watch('me', (me) => {
              if (me) {
                resolve(this.me);
                unwatch();
              }
            });
          }));
      }
      return Promise.resolve();
    },
    save(asDuplicate = false) {
      const doLoading = (status) => {
        if (asDuplicate) {
          this.loading.duplicate = status;
        } else {
          this.loading.save = status;
        }
      };
      doLoading(true);

      this.loginIfNecessary()
        .then(() => {
          if (!this.me || !this.me[idKey]) throw new Error('You need to be logged in.');

          const bundle = this.$refs.dataGrouper.bundleJson();
          const myListsRef = db.child(`users:lists/${this.me[idKey]}`);

          // save to firebase
          if (this.listId && !asDuplicate) {
            // update
            const listRef = myListsRef.child(this.listId);
            return listRef.set({
              ...bundle,
              // _updated: moment().format(),
            })
              .then(() => listRef); // pass on the reference
          }
          // create
          return myListsRef.push({
            // _created: moment().format(),
            ...bundle,
          });
        })
        .then((ref) => {
          this.listId = ref.key;
          this.$refs.snackbar.open();

          doLoading(false);
          this.dirty = false;
        })
        .catch((err) => {
          console.error(err); // eslint-disable-line no-console

          doLoading(false);
        });
    },
  },
  created() {
    this.loadJson('/static/grouped.json');
  },
  components: {
    DataGrouper,
  },
};
</script>

<style lang="scss">
html,
body,
#app {
  height: 100%;
  margin: 0;
}
#app {
  display: flex;
  flex-direction: column;

  > aside {
    display: flex;
    align-items: center;
    justify-content: center;
    flex-grow: 1;
  }
}

.md-toolbar {
  > div {
    display: flex;
    align-items: center;

    &:first-child {
      width: 25%;
      margin-right: 4px;
    }
    &:last-child {
      flex: 1;
      margin-left: 8px;
    }
  }
  .md-title {
    white-space: nowrap;
  }
  .md-input-inline {
    width: auto;
    flex-grow: 1;
  }
}

.file-upload-button {
  position: relative;

  input[type=file] {
    position: absolute;
    top: 0;
    left: 0;
    font-size: 1000px;
    opacity: 0;
  }
}
</style>
