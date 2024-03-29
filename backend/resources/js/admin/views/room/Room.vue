<template>
  <v-container>
    <v-data-table :headers="headers" :items="rooms" sort-by="roomname" class="elevation-1">
      <template v-slot:top>
        <v-toolbar flat>
          <v-toolbar-title>部屋一覧</v-toolbar-title>
          <v-divider class="mx-4" inset vertical></v-divider>
          <v-spacer></v-spacer>
          <span>編集ボタンから時間割を確認できます。</span>

          <!-- 部屋データ編集ダイアログ -->
          <v-dialog v-model="editRoomForm.dialog" max-width="500px" persistent>
            <v-form ref="editRoomForm" v-model="editRoomForm.validation.valid" lazy-validation>
              <v-card class="headline grey darken-2 text-center pa-2">
                <v-card-title>
                  <span class="headline white--text">編集</span>
                </v-card-title>

                <v-card-text>
                  <v-container>
                    <!-- 背景画像 -->
                    <!-- <v-card-text class="pa-1 white--text">部屋デザイン</v-card-text>
                    <span class="red--text">*PNG型式で2160px × 1200pxのみ対応</span>
                    <input
                      type="file"
                      @change="inputImage"
                      accept="image/png"
                      class="pa-2 mt-2 mb-6"
                      style="overflow: hidden"
                      v-if="editRoomForm.dialog"
                    /> -->

                    <!-- 部屋名 -->
                    <v-card-text class="pa-1 white--text">部屋名</v-card-text>
                    <v-text-field
                      v-model="editRoomForm.data.name"
                      :rules="editRoomForm.validation.nameRules"
                      label="部屋名"
                      solo
                      rounded
                      class="pa-2"
                    ></v-text-field>

                    <!-- Slack Webhook URL -->
                    <v-card-text class="pa-1 white--text">Slack Webhook URL</v-card-text>
                    <v-text-field
                      v-model="editRoomForm.data.slack"
                      :rules="editRoomForm.validation.slackRules"
                      label="Slack Webhook URL"
                      solo
                      rounded
                      class="pa-2"
                    ></v-text-field>

                    <!-- 時間割 -->
                    <!-- <v-card-text class="pa-1 white--text">時間割</v-card-text>
                    <v-list-item v-for="(time, index) in editRoomForm.data.timetable" :key="index">
                      <v-text-field
                        v-model="time.separate"
                        label="コマ開始時刻"
                        readonly
                        solo
                        rounded
                        @click="editTimetable(index, time.separate)"
                      ></v-text-field>

                      <v-select
                        v-model="time.status"
                        :items="status"
                        label="状態"
                        solo
                        rounded
                        class="pa-2"
                      ></v-select>
                    </v-list-item> -->

                    <!-- 時間入力ダイアログ -->
                    <!-- <v-dialog v-model="timePicker.dialog" width="290px">
                      <v-time-picker
                        v-if="timePicker.dialog"
                        v-model="timePicker.time"
                        full-width
                        @click:minute="setTimetable()"
                      ></v-time-picker>
                    </v-dialog> -->
                  </v-container>
                </v-card-text>

                <v-card-actions>
                  <v-spacer></v-spacer>
                  <v-btn color="error" :loading="editRoomForm.loading" @click="close()">
                    キャンセル
                  </v-btn>
                  <v-btn color="success" :loading="editRoomForm.loading" @click="submit()">
                    保存
                  </v-btn>
                </v-card-actions>
              </v-card>
            </v-form>
          </v-dialog>
        </v-toolbar>
      </template>

      <template v-slot:[`item.actions`]="{ item }">
        <v-icon small @click="editRoom(item)">mdi-pencil</v-icon>
      </template>

      <template v-slot:no-data>
        <v-btn color="primary" @click="getRooms()">再読み込み</v-btn>
      </template>
    </v-data-table>
  </v-container>
</template>

<script>
import { OK } from '@/consts/status';

export default {
  head: {
    title() {
      return {
        inner: '部屋',
      };
    },
  },
  data() {
    return {
      rooms: [], // 部屋一覧
      status: [
        { text: '自習時間', value: 'study' },
        { text: '休憩時間', value: 'break' },
      ],
      timePicker: {
        dialog: false,
        index: null,
        time: null,
      },
      headers: [
        { text: '部屋名', value: 'name' },
        { text: '編集', value: 'actions', sortable: false, align: 'center' },
      ],
      editRoomForm: {
        dialog: false,
        loading: false,
        index: -1,
        data: {},
        background: null, // 背景画像データ
        validation: {
          valid: false,
          nameRules: [(v) => !!v || '部屋名は必須項目です。'],
          slackRules: [
            (v) =>
              !v ||
              /https?:\/\/[-_.!~*\'()a-zA-Z0-9;\/?:\@&=+\$,%#]+/g.test(v) ||
              'URLを入力してください。',
          ],
        },
      },
    };
  },
  methods: {
    /**
     * 部屋データの取得
     */
    getRooms: async function () {
      let response = await axios.get('/api/admin/rooms');
      this.rooms = response.data;
    },

    /**
     * 部屋データの編集
     *
     * @param {Object} room - 編集する部屋
     */
    editRoom: function (room) {
      this.editRoomForm.index = this.rooms.indexOf(room);
      this.editRoomForm.data = Object.assign({}, room);

      // 時間割データの最適化
      // オブジェクトを配列化
      let sortedTimetable = [];
      Object.keys(this.editRoomForm.data.timetable).forEach((key) => {
        sortedTimetable.push({ separate: key, status: this.editRoomForm.data.timetable[key] });
      });

      // 時刻の小さい順に並べ替える
      sortedTimetable.sort((a, b) => {
        let comparison = 0;
        if (a.separate > b.separate) {
          comparison = 1;
        } else if (a.separate < b.separate) {
          comparison = -1;
        }
        return comparison;
      });

      this.editRoomForm.data.timetable = sortedTimetable;

      this.editRoomForm.dialog = true;
    },

    /**
     * 画像の入力
     *
     * @param {Event} event - 入力イベント
     */
    inputImage: function (event) {
      this.editRoomForm.background = event.target.files[0];
    },

    /**
     * 時刻データの編集
     *
     * @param {Number} index - 時間割に対する時刻のインデックス（表示データの更新で使用）
     * @param {String} time - 時刻
     */
    editTimetable: function (index, time) {
      this.timePicker.index = index;
      this.timePicker.time = time;
      this.timePicker.dialog = true;
    },

    /**
     * 時刻データのセット
     */
    setTimetable: function () {
      // 入力されたデータをセット
      this.editRoomForm.data.timetable[this.timePicker.index].separate = this.timePicker.time;
      this.timePicker.dialog = false;
    },

    /**
     * 編集ダイアログのクローズ
     */
    close: function () {
      this.editRoomForm.dialog = false;
      this.editRoomForm.loading = false;
      this.$nextTick(() => {
        this.$refs.editRoomForm.reset();
        this.editRoomForm.index = -1;
      });
    },

    /**
     * 編集データの保存
     */
    submit: async function () {
      if (this.$refs.editRoomForm.validate()) {
        this.editRoomForm.loading = true;

        // 時間割データの最適化
        // 時刻の小さい順に並べ替える
        this.editRoomForm.data.timetable.sort((a, b) => {
          let comparison = 0;
          if (a.separate > b.separate) {
            comparison = 1;
          } else if (a.separate < b.separate) {
            comparison = -1;
          }
          return comparison;
        });

        let timetable = {};
        this.editRoomForm.data.timetable.forEach((data) => {
          timetable[data.separate] = data.status;
        });

        let input = new FormData();
        input.append('_method', 'patch');
        input.append('name', this.editRoomForm.data.name);
        input.append('slack', this.editRoomForm.data.slack);
        input.append('timetable', JSON.stringify(timetable));
        if (this.editRoomForm.background !== null) {
          input.append('background', this.editRoomForm.background);
        }

        // 部屋データ保存処理
        let response = await axios.post('/api/admin/rooms/' + this.editRoomForm.data.id, input);

        if (response.status === OK) {
          if (this.editRoomForm.index > -1) {
            this.rooms[this.editRoomForm.index].name = this.editRoomForm.data.name;
            this.rooms[this.editRoomForm.index].slack = this.editRoomForm.data.slack;
            this.rooms[this.editRoomForm.index].timetable = timetable;
          } else {
            this.rooms.push(this.editRoomForm);
          }

          this.close();
        } else {
          this.editRoomForm.loading = false;
        }
      }
    },
  },
  created() {
    this.getRooms();
  },
};
</script>
