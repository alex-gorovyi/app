<template>
  <div class="interface-wysiwyg-container" ref="parent">
    <div ref="editor" :class="['interface-wysiwyg', readonly ? 'readonly' : '']"></div>
    <portal to="modal" v-if="chooseExisting">
      <v-modal
        :title="$t('choose_one')"
        :buttons="{
          done: {
            text: $t('done')
          }
        }"
        @close="chooseExisting = false"
        @done="chooseExisting = false"
      >
        <v-items
          collection="directus_files"
          view-type="cards"
          :selection="[]"
          :view-options="viewOptions"
          @select="insertItem($event[0])"
        ></v-items>
      </v-modal>
    </portal>
  </div>
</template>

<script>
import Quill from "quill";
import "quill/dist/quill.core.css";
import "./quill.theme.css";
import { ImageUpload } from "quill-image-upload";

Quill.register("modules/imageUpload", ImageUpload);

import mixin from "@directus/extension-toolkit/mixins/interface";

export default {
  name: "interface-wysiwyg",
  mixins: [mixin],
  mounted() {
    this.init();
  },
  watch: {
    value(newVal) {
      if (newVal !== this.editor.root.innerHTML) {
        this.editor.clipboard.dangerouslyPasteHTML(this.value);
      }
    }
  },
  data() {
    return {
      chooseExisting: false,
      viewOptions: {
        title: "title",
        subtitle: "type",
        content: "description",
        src: "data"
      }
    };
  },
  methods: {
    init() {
      let uploadURL = null;

      if (this.options.upload_files) {
        uploadURL = `${this.$store.state.auth.url}/${this.$store.state.auth.project}/files`;
      }

      this.editor = new Quill(this.$refs.editor, {
        theme: "snow",
        readOnly: this.readonly,
        modules: {
          toolbar:
            typeof this.options.toolbarOptions === "string"
              ? JSON.parse(this.options.toolbarOptions)
              : this.options.toolbarOptions,
          imageUpload: {
            // server url. If the url is empty then the base64 returns
            url: uploadURL,

            // custom form name
            name: "image",
            withCredentials: false,
            headers: {
              Authorization: `Bearer ${this.$api.token}`
            },
            csrf: { token: "token", hash: "" }, // add custom CSRF
            customUploader: null, // add custom uploader
            // personalize successful callback and call next function to insert new url to the editor
            callbackOK: (serverResponse, next) => {
              this.$store.dispatch("loadingFinished", "uploadingFile");
              // pass image url to editor
              if (typeof serverResponse === "string") {
                return next(serverResponse);
              } else if (this.options.custom_url) {
                return next(`${this.options.custom_url}${serverResponse.data.filename}`);
              }

              return next(serverResponse.data.data.full_url);
            },
            // personalize failed callback
            callbackKO: serverError => {
              this.$store.dispatch("loadingFinished", "uploadingFile");

              try {
                alert(JSON.parse(serverError.body).error.message);
              } catch (e) {
                console.error(e); // eslint-disable-line
              }
            },
            // optional
            // add callback when a image have been chosen
            checkBeforeSend: (file, next) => {
              this.$store.dispatch("loadingStart", {
                id: "uploadingFile"
              });
              next(file); // go back to component and send to the server
            }
          }
        }
      });

      this.editor.clipboard.dangerouslyPasteHTML(this.value);

      this.editor.on("text-change", () => {
        this.$emit("input", this.editor.root.innerHTML);
      });

      // Make custom icons for image buttons
      const customButton = this.$refs.parent.querySelector(".ql-choose-existing");
      if (customButton) {
        customButton.className += " material-icons icon";
        customButton.addEventListener("click", () => this.openModal());
      }
      const imageButton = this.$refs.parent.querySelector(".ql-image");
      if (imageButton) {
        imageButton.innerHTML = "";
        imageButton.className += " material-icons icon";
      }
    },
    openModal() {
      this.chooseExisting = true;
    },
    insertItem(image) {
      let url = image.data.full_url;
      if (this.options.custom_url) url = `${this.options.custom_url}${image.filename}`;
      const index = (this.editor.getSelection() || {}).index || this.editor.getLength();
      this.editor.insertEmbed(index, "image", url, "user");
      this.chooseExisting = false;
    }
  }
};
</script>

<style lang="scss">
.ql-container {
  font-size: 14px;
}

.ql-choose-existing {
  padding: 3px 5px;
  color: var(--light-gray);
  &:hover {
    color: var(--darkest-gray);
  }
  &:after {
    content: "collections";
    font-size: 20px;
  }
}

.ql-image {
  padding: 3px 5px;
  color: var(--light-gray);
  &:hover {
    color: var(--darkest-gray);
  }
  &:after {
    content: "add_photo_alternate";
    font-size: 20px;
  }
}

.ql-editor {
  &.readonly {
    background-color: var(--lightest-gray) !important;
    cursor: not-allowed;
    &:focus {
      color: var(--gray);
    }
  }
}
</style>

<style lang="scss" scoped>
.interface-wysiwyg-container {
  max-width: var(--width-x-large);
}

.material-icons {
  font-size: 20px;
}
</style>
