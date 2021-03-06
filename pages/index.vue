<template>
  <div class="row w-100 h-100">
    <div class="col-7 row index-left d-flex align-items-center m-0 p-0">
      <div class="col-12 index-left-logo d-flex align-items-center justify-content-between">
        <div><Logo /></div>
        <ToolButtons @createNewMemo="createNewMemo(true)" @importMD="importMD" @exportMD="exportMD" @importPDF="importPDF" />
      </div>
      <div class="col-12 index-left-textarea"><Editor ref="editor" /></div>
    </div>
    <div class="col-5 index-right m-0 p-0">
      <div class="col-12 h-100">
        <span ref="alttext" class="position-absolute index-right-alttext">Your PDF will be shown here</span>
        <iframe ref="viewer" width="100%" height="100%"></iframe>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue';
// eslint-disable-next-line import/named
import { fileOpen } from 'browser-fs-access';
import { indexData, opts } from '../types/index';

export default Vue.extend({
  data(): any {
    return {
      iframe: null,
      altTextElement: null,
      dbVersion: 1,
      dbName: '',
      storeName: '',
    } as indexData;
  },
  async mounted() {
    this.dbName = 'fileHandleDB';
    this.storeName = 'fileHandleStore';
    const openReq: IDBOpenDBRequest = indexedDB.open(this.dbName, this.dbVersion);
    openReq.onupgradeneeded = (event) => {
      if (!event.target) return;
      const db: any = event.target.result;
      db.createObjectStore(this.storeName, { keyPath: 'type' });
      console.log(`${this.storeName} was successfully created`);
    };
    openReq.onerror = (err: Event) => { throw new Error(`Open request against ${this.dbName} is blocked: ${err}`); };
    openReq.onsuccess = (event) => {
      if (!event.target) return;
      const db = event.target.result;
      this.dbVersion = db.version;
      const readTransaction = db.transaction(this.storeName, 'readonly');
      const store = readTransaction.objectStore(this.storeName);
      // PDFをIndexedDBから取得して表示
      const pdfCallback = (pdfHandle: any) => {
        this.insertPDF(pdfHandle);
      };
      this.getFileHandleFromDB(store, 'pdf', pdfCallback);

      db.close();
    };
    await this.$nextTick();
    const iframeElement: HTMLIFrameElement | null = this.$refs.viewer as HTMLIFrameElement;
    this.iframe = iframeElement;
    const alttext: HTMLElement = this.$refs.alttext as HTMLElement;
    this.altTextElement = alttext;

    window.addEventListener('beforeunload', () => {
      const text: string = this.$refs.editor.getValue();
      localStorage.setItem('mdText', text);
    });
    window.addEventListener('keydown', (event) => {
      if (event.key === 's' && event.metaKey) {
        event.preventDefault();
        this.exportMD();
      }
    });
  },
  methods: {
    getFileHandleFromDB(store: any, key: string, callback?: (fileHandle: any) => void): any {
      const getRequest = store.get(key);
      getRequest.onsuccess = (event: Event) => {
        if (!event.target || !event.target.result) return;
        const fileHandle: any = event.target.result.value;
        if (!callback) return;
        callback(fileHandle);
      };
      getRequest.onerror = (err: string) => {
        throw new Error(err);
      };
    },
    putDataToDB(key: string, value: any) {
      const data: {type: string, value: any} = { type: key, value };
      const openReqForPuttingData: IDBOpenDBRequest = indexedDB.open(this.dbName);
      openReqForPuttingData.onerror = (err) => {
        throw new Error(`Open request against ${this.dbName} is blocked: ${err}`);
      };
      openReqForPuttingData.onsuccess = (event: Event) => {
        if (!event.target) return;
        const db: any = event.target.result;
        const transaction = db.transaction(this.storeName, 'readwrite');
        const store = transaction.objectStore(this.storeName);
        const putReq = store.put(data);
        putReq.onsuccess = () => {
          console.log('Data was successfully added to DB');
          db.close();
        };
        transaction.oncomplete = () => {
          console.log('Transaction completed');
        };
      };
    },
    insertPDF(pdfFile: any) {
      const blobURL: string = window.URL.createObjectURL(pdfFile);
      if (!this.iframe) throw new Error('this.iframe not found');
      this.iframe.src = blobURL;
      this.altTextElement?.remove();
      this.putDataToDB('pdf', pdfFile);
    },
    createNewMemo(doesClearEditor: boolean = true) {
      if (doesClearEditor) this.$refs.editor.clear();
      const openReqForDeletingData: IDBOpenDBRequest = indexedDB.open(this.dbName);
      openReqForDeletingData.onerror = (err) => {
        throw new Error(`Open request against ${this.dbName} is blocked: ${err}`);
      };
      openReqForDeletingData.onsuccess = (event: Event) => {
        if (!event.target) return;
        const db: any = event.target.result;
        const transaction = db.transaction(this.storeName, 'readwrite');
        const store = transaction.objectStore(this.storeName);
        const deleteFileHandleReq = store.delete('mdFileHandle');
        deleteFileHandleReq.onsuccess = () => {
          console.log('MD FileHandle deleted');
          db.close();
        };
      };
      localStorage.setItem('mdText', '');
      localStorage.setItem('doesMDFileHandleExists', 'false');
      if (doesClearEditor) console.log('New memo has been created');
    },
    async importPDF() {
      const opts: opts = {
        mimeTypes: ['application/pdf'],
        multiple: false,
        description: 'PDF File'
      };
      try {
        const pdfFile = await fileOpen(opts);
        this.insertPDF(pdfFile);
      } catch (err) {
        console.log('Failed to import PDF: ', err);
      }
    },
    async importMD() {
      const opts = {
        types: [
          {
            description: 'Markdown',
            accept: { 'text/markdown': ['.md'] },
          },
        ],
        multiple: false,
      };
      try {
        let fileHandle: any;
        [fileHandle] = await window.showOpenFilePicker(opts); // eslint-disable-line prefer-const
        const mdFile: any = await fileHandle.getFile();
        const mdText: string = await mdFile.text();
        this.editor.setValue(mdText);
        localStorage.setItem('mdText', mdText);
        this.putDataToDB('mdFileHandle', fileHandle);
        localStorage.setItem('doesMDFileHandleExists', 'true');
      } catch (err) {
        console.log(err);
      }
    },
    async exportMD() {
      const expOpts = {
        suggestedName: 'memo.md',
        types: [
          {
            description: 'Markdown',
            accept: { 'text/markdown': ['.md'], },
          },
        ],
      };
      const editorText: string = this.$refs.editor.getValue();
      const doesMDFileHandleExists: string | null = localStorage.getItem('doesMDFileHandleExists');
      // FileHandleが見つからなかった場合はドキュメントをダウンロード
      if (doesMDFileHandleExists !== 'true') {
        try {
          const fileHandle = await window.showSaveFilePicker(expOpts);
          const writable: any = await fileHandle.createWritable();
          await writable.write(editorText);
          await writable.close();
          this.putDataToDB('mdFileHandle', fileHandle);
          localStorage.setItem('doesMDFileHandleExists', 'true');
        } catch (err) {
          console.log(err);
        }
        return;
      }
      // FileHandleがある場合は上書き
      const exportCallBack = async (mdHandle: any) => {
        const fileHandle = mdHandle;
        const writable: any = await fileHandle.createWritable();
        await writable.write(editorText);
        await writable.close();
      };
      const openReq: IDBOpenDBRequest = indexedDB.open(this.dbName);
      openReq.onsuccess = (event: Event) => {
        if (!event.target) return;
        const db: any = event.target.result;
        const transaction = db.transaction(this.storeName, 'readonly');
        const store = transaction.objectStore(this.storeName);
        this.getFileHandleFromDB(store, 'mdFileHandle', exportCallBack);
        db.close();
      };
    }
  }
});
</script>

<style lang="scss">
.index-left {
  height: calc(100vh - 20px);
  &-logo {
    margin-left: 10px;
    height: 60px;
  }
  &-textarea {
    margin-left: 10px;
    height: calc(100% - 60px);
  }
}
.index-right {
  @extend %completely-center;

  position: relative;
  &-alttext {
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-family: $generic-fonts-inter;
    color: $black;
  }
  iframe {
    width: 100%;
    margin: 5px 0;
    height: calc(100% - 10px);
    border: none;
  }
}
</style>
