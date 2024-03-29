<template>
  <v-container>
    <v-form
      ref="commentForm"
      v-model="commentForm.validation.valid"
      lazy-validation
      id="comment-form"
      class="mx-auto my-6"
    >
      <v-textarea
        v-model="commentForm.body"
        :rules="commentForm.validation.bodyRules"
        :maxlength="commentForm.max"
        :disabled="commentForm.loading"
        append-icon="mdi-send"
        placeholder="コメント"
        counter
        solo
        auto-grow
        rows="1"
        @click:append="submit()"
      ></v-textarea>
    </v-form>

    <v-container v-if="comments.length">
      <v-card class="my-2" v-for="comment in comments" :key="comment.id">
        <v-card-actions class="d-block">
          <v-row no-gutters justify="end" v-if="comment.user.id === authUser.id">
            <v-btn icon x-small @click="deleteComment(comment)">
              <v-icon>mdi-close</v-icon>
            </v-btn>
          </v-row>

          <!-- 内容 -->
          <pre class="ml-3 text-body-1 text-left" v-html="$formatStr(comment.body)"></pre>

          <!-- 投稿日時 -->
          <p class="mt-6 mb-0 mr-4 text-right text-body-2">
            {{ $moment(comment.created_at).format('MM/DD HH:mm') }}
          </p>
        </v-card-actions>

        <v-row class="mt-3">
          <v-col
            cols="8"
            class="pointer"
            @click="
              $store.dispatch('dialog/open', { type: 'user', username: comment.user.username })
            "
          >
            <v-row>
              <!-- ユーザーアイコン -->
              <v-col cols="3" class="my-auto text-center">
                <v-avatar
                  size="40"
                  :style="{ 'box-shadow': '0 0 0 3px ' + $statusColor(comment.user.status) }"
                >
                  <img :src="$storage('icon') + comment.user.icon" />
                </v-avatar>
              </v-col>

              <!-- ユーザー名 -->
              <v-col cols="9" class="my-auto text-start">
                <p class="mb-0 text-body-1 text-truncate">{{ comment.user.handlename }}</p>
                <p class="mb-0 text-body-2 text-truncate">@{{ comment.user.username }}</p>
              </v-col>
            </v-row>
          </v-col>

          <v-col cols="3" class="my-auto text-right">
            <!-- いいねボタン -->
            <v-btn
              icon
              :color="comment.favorite_id_by_auth_user ? 'red' : 'gray'"
              @click="favorite(comment)"
            >
              <v-icon>mdi-heart</v-icon>
              <span>{{ comment.favorites_count }}</span>
            </v-btn>
          </v-col>

          <v-spacer></v-spacer>
        </v-row>
      </v-card>
    </v-container>

    <!-- コメント削除確認ダイアログ -->
    <v-dialog v-model="deleteForm.dialog" max-width="500px" persistent>
      <v-card class="headline grey darken-2 text-center pa-2">
        <v-card-title>
          <span class="headline white--text">削除しますか？</span>
        </v-card-title>

        <v-card-text>
          <v-container>
            <v-card-text class="pa-1 white--text">
              {{ deleteForm.data.body }}
            </v-card-text>
          </v-container>
        </v-card-text>

        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="grey" :loading="deleteForm.loading" @click="deleteSubmit()">削除</v-btn>
          <v-btn color="error" :loading="deleteForm.loading" @click="deleteForm.dialog = false">
            キャンセル
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script>
import { OK } from '@/consts/status';

export default {
  props: {
    itemType: String, // カルテか投稿どちらのコメントか
    itemId: Number, // コメントが紐付いているアイテムのID
    comments: Array, // コメント一覧
  },

  data() {
    return {
      commentForm: {
        body: '', // 内容
        max: 1000, // 最大長
        loading: false,
        validation: {
          valid: false,
          bodyRules: [(v) => !!v || '内容が無いようです。'],
        },
      },
      deleteForm: {
        dialog: false,
        loading: false,
        data: {},
      },
    };
  },

  computed: {
    authUser() {
      return this.$store.getters['auth/user'];
    },
  },

  methods: {
    /**
     * コメントの投稿
     */
    submit: async function () {
      if (this.$refs.commentForm.validate()) {
        this.commentForm.loading = true;

        let response;
        if (this.itemType === 'karte') {
          response = await axios.post('/api/comments', {
            karte_id: this.itemId,
            body: this.commentForm.body,
          });
        } else if (this.itemType === 'post') {
          response = await axios.post('/api/comments', {
            post_id: this.itemId,
            body: this.commentForm.body,
          });
        }

        if (response.status === OK) {
          // コメントデータの更新
          this.$emit('update');
          this.$refs.commentForm.reset();
        }

        this.commentForm.loading = false;
      }
    },

    /**
     * コメントの削除
     *
     * @param {Object} comment - 削除するコメント
     */
    deleteComment: function (comment) {
      this.deleteForm.data = comment;
      this.deleteForm.dialog = true;
    },

    /**
     * 削除データの送信
     */
    deleteSubmit: async function () {
      this.deleteForm.loading = true;

      let response = await axios.delete('/api/comments/' + this.deleteForm.data.id);

      if (response.status === OK) {
        // コメントデータの更新
        this.$emit('update');
        this.deleteForm.dialog = false;
      }

      this.deleteForm.loading = false;
    },

    /**
     * いいね処理
     *
     * @param {Object} comment - いいねするコメント
     */
    favorite: async function (comment) {
      if (!comment.favorite_id_by_auth_user) {
        // いいね処理
        let response = await axios.post('/api/favorites', { comment_id: comment.id });

        if (response.status === OK) {
          // IDの追加とカウントアップ
          comment.favorite_id_by_auth_user = response.data;
          comment.favorites_count += 1;
        }
      } else {
        // いいね解除処理
        let response = await axios.delete('/api/favorites/' + comment.favorite_id_by_auth_user);

        if (response.status === OK) {
          // IDの削除とカウントダウン
          comment.favorite_id_by_auth_user = null;
          comment.favorites_count -= 1;
        }
      }
    },
  },
};
</script>

<style lang="scss" scoped>
pre {
  white-space: pre-wrap;
}

.pointer {
  cursor: pointer;
}
</style>
