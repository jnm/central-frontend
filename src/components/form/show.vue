<!--
Copyright 2017 ODK Central Developers
See the NOTICE file at the top-level directory of this distribution and at
https://github.com/opendatakit/central-frontend/blob/master/NOTICE.

This file is part of ODK Central. It is subject to the license terms in
the LICENSE file found in the top-level directory of this distribution and at
https://www.apache.org/licenses/LICENSE-2.0. No part of ODK Central,
including this file, may be copied, modified, propagated, or distributed
except according to the terms contained in the LICENSE file.
-->
<template>
  <div>
    <form-head @fetch-draft="fetchDraft"/>
    <page-body>
      <loading :state="initiallyLoading"/>
      <div v-show="dataExists">
        <!-- We only include SubmissionList, because it is the only component
        whose state we want to preserve when the user navigates to a different
        tab. -->
        <keep-alive include="SubmissionList">
          <!-- <router-view> is immediately created and can send its own
          requests even before the server has responded to the requests from
          ProjectHome and FormShow. -->
          <router-view v-bind="routerViewProps" @fetch-form="fetchForm"
            @fetch-draft="fetchDraft"/>
        </keep-alive>
      </div>
    </page-body>
  </div>
</template>

<script>
import FormHead from './head.vue';
import Loading from '../loading.vue';
import Option from '../../util/option';
import PageBody from '../page/body.vue';
import SubmissionList from '../submission/list.vue';
import { apiPaths } from '../../util/request';
import { noop } from '../../util/util';

const REQUEST_KEYS = ['project', 'form', 'formDraft', 'attachments'];

export default {
  name: 'FormShow',
  components: { FormHead, Loading, PageBody },
  props: {
    projectId: {
      type: String,
      required: true
    },
    xmlFormId: {
      type: String,
      required: true
    }
  },
  data() {
    return {
      // Passing these to SubmissionList in order to facilitate SubmissionList
      // testing.
      submissionChunkSizes: SubmissionList.props.chunkSizes.default(),
      scrolledToBottom: SubmissionList.props.scrolledToBottom.default
    };
  },
  computed: {
    initiallyLoading() {
      return this.$store.getters.initiallyLoading(REQUEST_KEYS);
    },
    dataExists() {
      return this.$store.getters.dataExists(REQUEST_KEYS);
    },
    routerViewProps() {
      switch (this.$route.name) {
        case 'FormOverview':
          return { projectId: this.projectId, xmlFormId: this.xmlFormId };
        case 'SubmissionList':
          return {
            projectId: this.projectId,
            xmlFormId: this.xmlFormId,
            chunkSizes: this.submissionChunkSizes,
            scrolledToBottom: this.scrolledToBottom
          };
        default:
          return {};
      }
    }
  },
  watch: {
    xmlFormId() {
      this.fetchData();
    }
  },
  created() {
    this.fetchData();
  },
  methods: {
    fetchForm() {
      this.$store.dispatch('get', [{
        key: 'form',
        url: apiPaths.form(this.projectId, this.xmlFormId),
        extended: true,
        success: ({ form, submissionsChunk }) => {
          if (submissionsChunk == null) return;
          if (submissionsChunk['@odata.count'] === form.submissions)
            return;
          this.$store.commit('setData', {
            key: 'form',
            value: form.with({
              submissions: submissionsChunk['@odata.count']
            })
          });
        }
      }]).catch(noop);
    },
    reconcileFormDraftAndAttachments({ formDraft, attachments }) {
      if (formDraft == null || attachments == null) return;
      if (formDraft.isDefined() && attachments.isEmpty()) {
        this.$store.commit('setData', {
          key: 'formDraft',
          value: Option.none()
        });
      } else if (formDraft.isEmpty() && attachments.isDefined()) {
        this.$store.commit('setData', {
          key: 'attachments',
          value: Option.none()
        });
      }
    },
    fetchDraft() {
      this.$store.dispatch('get', [
        {
          key: 'formDraft',
          url: apiPaths.formDraft(this.projectId, this.xmlFormId),
          extended: true,
          fulfillProblem: ({ code }) => code === 404.1,
          success: this.reconcileFormDraftAndAttachments
        },
        {
          key: 'attachments',
          url: apiPaths.formDraftAttachments(this.projectId, this.xmlFormId),
          fulfillProblem: ({ code }) => code === 404.1,
          success: this.reconcileFormDraftAndAttachments
        }
      ]).catch(noop);
    },
    fetchData() {
      this.fetchForm();
      this.fetchDraft();
    }
  }
};
</script>
