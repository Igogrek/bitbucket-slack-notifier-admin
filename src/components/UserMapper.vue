<template>
  <div>
    <div class="card">
      <div class="card-table">
        <div class="content">
          <table class="table is-fullwidth is-striped">
            <thead>
            <tr>
              <td>BitBucket Id</td>
              <td>Slack Id</td>
              <td>Actions</td>
            </tr>
            </thead>
            <tbody>
            <tr>
              <td>
                <input class="input is-small" placeholder="BitBucket Id" v-model="newUser.bitBucketId">
              </td>
              <td>
                <input class="input is-small" placeholder="Slack Id" v-model="newUser.slackId">
              </td>
              <td>
                <a class="button is-small"
                   v-bind:disabled="!newUser.slackId"
                   @click="testSlackChannel(newUser)"
                   v-bind:class="{
                     'is-loading': newUser.slackLoading ,
                     'is-success': newUser.slackStatus == 'success',
                     'is-danger': newUser.slackStatus == 'error'
                   }">Test Slack ID</a>
                <a class="button is-small is-success"
                   v-bind:disabled="!newUser.bitBucketId || !newUser.slackId"
                   @click="addUserMapping">Add</a>
              </td>
            </tr>
            <tr v-for="user in users" v-bind:class="{ 'is-selected': user.unsaved }">
              <td>{{user.bitBucketId}}</td>
              <td>{{user.slackId}}</td>
              <td>
                <a class="button is-small test-slack"
                   @click="testSlackChannel(user)"
                   v-bind:class="{
                     'is-loading': user.slackLoading ,
                     'is-success': user.slackStatus == 'success',
                     'is-danger': user.slackStatus == 'error'
                   }">Test Slack ID</a>
                <a class="button is-small"
                   @click="testUserMapping(user)"
                   v-if="!user.unsaved"
                   v-bind:class="{
                     'is-loading': user.loading ,
                     'is-success': user.status == 'success',
                     'is-danger': user.status == 'error'
                   }">
                  Test Mapping
                </a>
                <a class="button is-small is-danger"
                   @click="removeUserMapping(user)">
                  Remove
                </a>
              </td>
            </tr>
            </tbody>
          </table>
        </div>
      </div>
      <footer class="card-footer">
        <a href="#" class="card-footer-item" @click="saveMappings">Save User Mappings</a>
      </footer>
    </div>
  </div>
</template>

<script>
  import axios from 'axios';
  import testMessage from './test-message.json';

  export default {
    name: 'UserMapper',
    data() {
      return {
        newUser: {
          bitBucketId: '',
          slackId: '',
          status: ''
        },
        users: []
      }
    },
    created() {
      axios.get(`/users`)
        .then(response => {
          this.users = this.usersToKeyValues(response.data)
        })
    },
    methods: {
      usersToKeyValues(userMap) {
        const keyValuePairs = [];
        for (const key of Object.keys(userMap)) {
          keyValuePairs.push({bitBucketId: key, slackId: userMap[key]});
        }
        return keyValuePairs;
      },
      testUserMapping(user) {
        this.$set(user, 'loading', true);
        testMessage.pullRequestUrl = location.href;
        testMessage.pullRequestReviewers = user.bitBucketId;

        axios.post(`/notify`, testMessage)
          .then(response => {
            this.$set(user, 'status', 'success');
            this.$set(user, 'loading', false);
          })
          .catch(error => {
            this.$set(user, 'status', 'error');
            this.$set(user, 'loading', false);
          });
      },
      testSlackChannel(user) {
        this.$set(user, 'slackLoading', true);
        axios.post(`/notify/test`, {channel: user.slackId, url: location.href})
          .then(response => {
            this.$set(user, 'slackStatus', 'success');
            this.$set(user, 'slackLoading', false);
          })
          .catch(error => {
            this.$set(user, 'slackStatus', 'error');
            this.$set(user, 'slackLoading', false);
          });
      },
      addUserMapping() {
        if (this.newUser.slackId && this.newUser.bitBucketId) {
          this.users.push(Object.assign({unsaved: true}, this.newUser));
          this.newUser.slackId = '';
          this.newUser.bitBucketId = '';
          this.newUser.status = '';
        }
      },
      removeUserMapping(user) {
        this.users.splice(this.users.indexOf(user), 1);
      },
      saveMappings() {
        const mapping = {};
        this.users.forEach(user => {
          mapping[user.bitBucketId] = user.slackId;
        });
        axios.post(`/users`, mapping)
          .then(response => {
            this.users.forEach(user => {
              user.unsaved = false;
            });
          })
      }
    }
  }
</script>

<style scoped lang="scss">
  @import '../../node_modules/bulma/sass/utilities/initial-variables';

  $light-orange: lighten($orange, 20%);

  .table tr.is-selected {
    background-color: $light-orange;

    .test-slack {
      color: $black;
    }

    &:hover {
      background-color: $light-orange;
    }
  }
</style>
