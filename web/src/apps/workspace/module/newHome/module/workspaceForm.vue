<template>
  <Modal
    v-model="ProjectShow"
    :title="actionType === 'add' ? $t('message.workspace.newWorkspace') : $t('message.workspace.editorWorkspace')"
    :closable="false">
    <Form
      :label-width="100"
      label-position="left"
      ref="projectForm"
      :model="projectDataCurrent"
      :rules="formValid"
      v-if="ProjectShow">
      <FormItem
        :label="$t('message.workspace.workName')"
        prop="name">
        <Input
          v-model="projectDataCurrent.name"
          :placeholder="$t('message.workspace.enterName')"
          :disabled="actionType === 'modify'" />
      </FormItem>
      <FormItem
        :label="$t('message.workspace.department')"
        prop="department">
        <Select
          v-model="projectDataCurrent.department"
          :placeholder="$t('message.workspace.selectDepartment')">
          <Option
            v-for="(item, index) in departments"
            :label="item.name"
            :value="String(item.id)"
            :key="index"/>
        </Select>
      </FormItem>
      <FormItem
        :label="$t('message.workspace.label')"
        prop="label">
        <we-tag
          :new-label="$t('message.workspace.addLabel')"
          :tag-list="projectDataCurrent.label"
          @add-tag="addTag"
          @delete-tag="deleteTag"></we-tag>
      </FormItem>
      <FormItem
        :label="$t('message.workspace.description')"
        prop="description">
        <Input
          v-model="projectDataCurrent.description"
          type="textarea"
          :placeholder="$t('message.workspace.pleaseInputWorkspaceDesc')" />
      </FormItem>
    </Form>
    <div slot="footer">
      <Button
        type="text"
        size="large"
        @click="Cancel">{{$t('message.workspace.cancel')}}</Button>
      <Button
        type="primary"
        size="large"
        @click="Ok">{{$t('message.workspace.ok')}}</Button>
    </div>
  </Modal>
</template>
<script>
import tag from '@/components/tag/index.vue';
import { GetDepartments, CheckWorkspaceNameExist } from '@/common/service/apiCommonMethod.js';
export default {
  components: {
    'we-tag': tag,
  },
  props: {
    projectData: {
      type: Object,
      default: null,
    },
    addProjectShow: {
      type: Boolean,
      default: false,
    },
    actionType: {
      type: String,
      default: '',
    },
  },
  data() {
    return {
      ProjectShow: false,
      departments: [],
    };
  },
  computed: {
    projectDataCurrent() {
      return this.projectData;
    },
    formValid() {
      return {
        name: [
          { required: true, message: this.$t('message.workspace.enterName'), trigger: 'blur' },
          { message: `${this.$t('message.workspace.nameLength')}128`, max: 128 },
          { type: 'string', pattern: /^[a-zA-Z][a-zA-Z0-9_]*$/, message: this.$t('message.workspace.validNameDesc'), trigger: 'blur' },
          { validator: this.checkNameExist, message: this.$t('message.workspace.validNameExist'), trigger: 'blur' },
        ],
        description: [
          { required: true, message: this.$t('message.workspace.pleaseInputWorkspaceDesc'), trigger: 'blur' },
        ],
        department: [
          { required: true, message: this.$t('message.workspace.selectDepartment'), trigger: 'change' },
        ],
      }
    }
  },
  mounted() {
    GetDepartments().then((res) => {
      this.departments = res.departments;
    });
  },
  watch: {
    addProjectShow(val) {
      this.ProjectShow = val;
    },
    ProjectShow(val) {
      this.$emit('show', val);
    },
  },
  methods: {
    checkNameExist(rule, value, callback) {
      CheckWorkspaceNameExist({ name: value }).then((res) => {
        if (res.workspaceNameExists) {
          callback(new Error('不可重复'));
        } else {
          callback();
        }
      });
    },
    Ok() {
      this.$refs.projectForm.validate((valid) => {
        if (valid) {
          this.$emit('confirm', this.projectDataCurrent);
          this.ProjectShow = false;
        } else {
          this.$Message.warning(this.$t('message.workspace.failedNotice'));
        }
      });
    },
    Cancel() {
      this.ProjectShow = false;
    },
    addTag(label) {
      if (this.projectDataCurrent.label) {
        this.projectDataCurrent.label += `,${label}`;
      } else {
        this.projectDataCurrent.label = label;
      }
    },
    deleteTag(label) {
      const tmpArr = this.projectDataCurrent.label.split(',');
      const index = tmpArr.findIndex((item) => item === label);
      tmpArr.splice(index, 1);
      this.projectData.label = tmpArr.toString();
    }
  },
};
</script>
