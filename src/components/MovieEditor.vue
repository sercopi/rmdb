<template>
  <q-dialog v-model="show" :persistent="true">
    <q-layout view="hHh lpR fFf" container class="layout bg-white">
      <q-header elevated class="bg-primary text-white">
        <q-toolbar class="toolbar">
          <q-btn
            flat
            icon="play_arrow"
            @click="playMovie"
            :disable="!hasFileAssociated"
          >
            <q-tooltip :delay="btnDelay" content-class="bg-accent">{{
              $t("movie_editor.play_btn")
            }}</q-tooltip>
          </q-btn>
          <q-btn flat icon="edit" @click="editBlocked = !editBlocked">
            <q-tooltip :delay="btnDelay" content-class="bg-accent">{{
              $t("movie_editor.edit_btn")
            }}</q-tooltip>
          </q-btn>
          <q-btn flat icon="delete" @click="confirmRemoveMovie = true">
            <q-tooltip :delay="btnDelay" content-class="bg-accent">{{
              $t("movie_editor.delete_btn")
            }}</q-tooltip>
          </q-btn>
          <q-btn
            flat
            icon="folder"
            @click="openMovieFolder"
            :disable="!hasFileAssociated"
          >
            <q-tooltip :delay="btnDelay" content-class="bg-accent">{{
              $t("movie_editor.open_location")
            }}</q-tooltip>
          </q-btn>
          <q-space />
          <q-btn flat icon="close" @click="closeEditor" v-close-popup />
        </q-toolbar>
      </q-header>

      <q-page-container>
        <q-page class="no-overflow bg-custom">
          <movieForm :movie="updatedMovie" :readonly="editBlocked" />

          <confirmDialog
            :dialogOpen.sync="confirmRemoveMovie"
            :title="
              $t('movie_editor.delete_dialog', {
                title: updatedMovie.title,
              })
            "
            :callBack="removeMovie"
          />
        </q-page>
      </q-page-container>

      <q-footer elevated class="bg-grey-8 text-white">
        <q-toolbar>
          <q-space />
          <q-btn :disable="editBlocked" flat icon="save" @click="updateMovie">
            <q-tooltip
              :delay="btnDelay"
              content-class="bg-accent"
              anchor="center left"
              self="center right"
              >{{ $t("movie_editor.save_btn") }}</q-tooltip
            >
          </q-btn>
        </q-toolbar>
      </q-footer>
      <q-dialog v-model="infoDialogOpen" position="bottom">
        <q-card style="width: 350px">
          <q-card-section class="row items-center no-wrap">
            <span>{{ infoDialogContent }}</span>
          </q-card-section>
        </q-card>
      </q-dialog>
    </q-layout>
  </q-dialog>
</template>

<script>
import Movie from "@/interfaces/movie.interface";
import MovieForm from "@/components/MovieForm";
import ConfirmDialog from "@/components/ConfirmDialog";
import { movieDatabase } from "@/database";
import { openByPath, checkPathIsCorrect } from "@/movieFilePath.service";

export default {
  name: "MovieEditor",
  data() {
    return {
      show: true,
      updatedMovie: {},
      editBlocked: true,
      hasFileAssociated: false,
      confirmRemoveMovie: false,
      btnDelay: 1000,
      infoDialogOpen: false,
      infoDialogContent: "",
    };
  },
  mounted() {
    this.updatedMovie = { ...this.movie };
    this.updatedMovie.posterUrl = { ...this.updatedMovie.posterUrl };

    this.hasFileAssociated = this.updatedMovie.filePath !== "";
  },
  methods: {
    closeEditor() {
      this.$emit("close");
    },
    playMovie() {
      const error = openByPath(this.updatedMovie.filePath);
      if (error)
        this.showInfoDialog(this.$t("common.error_movie_file_path"));
    },
    openMovieFolder() {
      const filePathArray = this.updatedMovie.filePath.split(/[\\/]+/);
      filePathArray.pop();
      const folderPath = filePathArray.join("/");

      const error = openByPath(folderPath);
      if (error)
        this.showInfoDialog(this.$t("common.error_movie_file_path"));
    },
    removeMovie() {
      movieDatabase
        .deleteMovie(this.movie)
        .then(() => {
          this.$emit("remove", this.movie.id);
          this.closeEditor();
        })
        .catch(console.log);
    },
    async updateMovie() {
      const movieExist = await movieDatabase.movieExist(this.updatedMovie);

      if (movieExist) {
        this.showInfoDialog(this.$t("common.error_movieAlreadyExists"));
        return;
      }

      const newPath = checkPathIsCorrect(this.updatedMovie.filePath);
      this.updatedMovie.filePath = newPath;

      if (!newPath) {
        this.showInfoDialog(this.$t("common.error_movie_file_path"));
        return;
      }

      movieDatabase
        .updateMovie(
          this.updatedMovie,
          this.movie.posterUrl.big !== this.updatedMovie.posterUrl.big ||
            this.movie.title !== this.updatedMovie.title
        )
        .then(() => {
          this.$emit("update", this.updatedMovie);
          this.hasFileAssociated = this.updatedMovie.filePath !== "";
          this.showInfoDialog(this.$t("movie_editor.update_successful"));
          this.editBlocked = true;
        })
        .catch(console.log);
    },
    showInfoDialog(content) {
      this.infoDialogOpen = true;
      this.infoDialogContent = content;

      setTimeout(() => {
        this.infoDialogOpen = false;
      }, 3000);
    },
  },
  props: {
    movie: Movie,
  },
  components: {
    MovieForm,
    ConfirmDialog,
  },
};
</script>

<style scoped>
.layout {
  max-width: 1000px;
  width: 1000px;
  height: 600px;
}

.toolbar > * {
  margin-left: 0.5em;
}

.no-overflow {
  overflow: hidden;
}
</style>
