<template>
  <layout-content :header="$t('commons.form.detail')" :back-to="{name: 'Pods'}" v-loading="loading">
    <div v-if="!yamlShow">
      <el-row :gutter="20" class="row-box">
        <el-col :span="12">
          <el-card class="el-card">
            <table style="width: 100%" class="myTable">
              <tr>
                <th scope="col" align="left">
                  <h3>{{ $t("business.common.basic") }}</h3>
                </th>
                <th scope="col"></th>
              </tr>
              <tr>
                <td>{{ $t("commons.table.name") }}</td>
                <td colspan="4">{{ form.metadata.name }}</td>
              </tr>
              <tr>
                <td>{{ $t("business.namespace.namespace") }}</td>
                <td colspan="4">{{ form.metadata.namespace }}</td>
              </tr>
              <tr>
                <td>Pod IP</td>
                <td colspan="4">{{ form.status.podIP }}</td>
              </tr>
              <tr>
                <td>{{ $t("business.cluster.nodes") }}</td>
                <td colspan="4">{{ form.spec.nodeName }}</td>
              </tr>
              <tr>
                <td>{{ $t("commons.table.created_time") }}</td>
                <td colspan="4">{{ form.metadata.creationTimestamp | age }}</td>
              </tr>
              <tr>
                <td>{{ $t("commons.table.status") }}</td>
                <td colspan="4">{{ $t("commons.status." + form.status.phase) }}</td>
              </tr>
              <tr >
                <td>{{ $t("business.workload.pod_whole_domain") }}</td>
                <td colspan="4" v-if="this.controller== 'StatefulSet' && this.pod_st_name != '' ">{{this.pod_st_name}}.{{this.form.metadata.namespace}}.svc.{{this.cluster_domain}}</td>
                <td colspan="4" v-else-if="this.ip!='' ">{{this.ip}}.pod.{{this.cluster_domain}}</td>
              </tr>
              <tr>
                <td>{{ $t("business.workload.pod_simple_domain") }}</td>
                <td colspan="4" v-if="this.controller== 'StatefulSet' && this.pod_st_name != '' ">{{this.pod_st_name}}.{{this.form.metadata.namespace}}.svc</td>
                <td colspan="4" v-else-if="this.ip!='' ">{{this.ip}}.pod</td>
              </tr>
            </table>
            <div class="bottom-button">
              <el-button @click="yamlShow=!yamlShow">{{ $t("commons.button.view_yaml") }}</el-button>
            </div>
          </el-card>
        </el-col>
        <el-col :span="12">
          <el-card class="el-card">
            <h3>{{ $t("business.common.conditions") }}</h3>
            <complex-table :data="form.status.conditions">
              <el-table-column :label="$t('business.pod.type')" prop="type" min-width="30" show-overflow-tooltip />
              <el-table-column :label="$t('commons.table.status')" prop="status" min-width="30" />
              <el-table-column :label="$t('business.workload.lastTransitionTime')" min-width="50">
                <template v-slot:default="{row}">
                  {{ row.lastTransitionTime | datetimeFormat }}
                </template>
              </el-table-column>
            </complex-table>
          </el-card>
        </el-col>
      </el-row>

      <h2 style="margin-top: 40px">{{$t('business.event.event')}}</h2>
      <complex-table :data="eventList">
        <el-table-column :label="$t('commons.table.type')" prop="type" min-width="20" show-overflow-tooltip fix />
        <el-table-column :label="$t('business.pod.reason')" prop="reason" min-width="20" show-overflow-tooltip fix />
        <el-table-column :label="$t('commons.table.name')" prop="metadata.creationTimestamp" show-overflow-tooltip min-width="45">
          <template v-slot:default="{row}">
            {{ row.metadata.creationTimestamp | datetimeFormat }}
          </template>
        </el-table-column>
        <el-table-column :label="$t('business.workload.source')" min-width="50">
          <template v-slot:default="{row}">
            <div v-for="(item, index) in row.source" :key="index">
              <el-tag type="success">{{item}}</el-tag>
            </div>
          </template>
        </el-table-column>
        <el-table-column :label="$t('business.pod.message')" prop="message" min-width="150" />
      </complex-table>

      <h2 style="margin-top: 40px">Spec</h2>
      <el-row :gutter="20" style="margin-top: 20px" class="row-box">
        <el-col :span="8">
          <el-card class="el-card">
            <ko-detail-general :yamlInfo="form" />
          </el-card>
        </el-col>
        <el-col :span="8">
          <el-card class="el-card">
            <ko-detail-volume :yamlInfo="form" />
          </el-card>
        </el-col>
        <el-col :span="8">
          <el-card class="el-card">
            <ko-detail-toleration :yamlInfo="form" />
          </el-card>
        </el-col>
      </el-row>

      <h2 style="margin-top: 40px">{{$t('business.workload.container')}}</h2>
      <el-row style="margin-top: 20px">
        <ko-detail-containers :yamlInfo="form" :cluster="clusterName" />
      </el-row>
    </div>
    <div v-if="yamlShow">
      <yaml-editor :value="sourceYaml" :read-only="true"></yaml-editor>
      <div class="bottom-button">
        <el-button @click="yamlShow=!yamlShow">{{ $t("commons.button.back_detail") }}</el-button>
      </div>
    </div>
  </layout-content>
</template>

<script>
import LayoutContent from "@/components/layout/LayoutContent"
import { getWorkLoadByName } from "@/api/workloads"
import YamlEditor from "@/components/yaml-editor"
import KoDetailContainers from "@/components/detail/detail-containers"
import KoDetailGeneral from "@/components/detail/pod/detail-general"
import KoDetailVolume from "@/components/detail/pod/detail-volume"
import KoDetailToleration from "@/components/detail/pod/detail-toleration"
import { listEventsWithPodSelector } from "@/api/events"
import ComplexTable from "@/components/complex-table"
import {get_cluster_domain} from "@/utils/cluster_domain"
import { getStatefulSet } from "@/api/statefulsets"
import { getService } from "@/api/services"
export default {
  name: "PodDetail",
  components: { LayoutContent, YamlEditor, ComplexTable, KoDetailGeneral, KoDetailContainers, KoDetailVolume, KoDetailToleration },
  props: {
    name: String,
    namespace: String,
  },
  data() {
    return {
      sourceYaml:{},
      form: {
        metadata: {},
        spec: {
          nodeName: "",
        },
        status: {
          phase: "Running",
          containerStatuses: [],
        },
      },
      yamlShow: false,
      activeName: "Containers",
      loading: false,
      clusterName: "",
      eventList: [],
      cluster_domain: "",
      pod_st_name:"",
      ip:"",
      controller: "",
      pod_st_name: "" ,
    }
  },
  methods: {
    async  getDetail() {
      this.loading = true
      const res=await getWorkLoadByName(this.clusterName, "pods", this.namespace, this.name)
      this.sourceYaml= structuredClone(res)
      this.form = res
      await this.loadEvents()
      await this.get_pod_domain()
      this.loading = false
    },
    async   loadEvents() {
      let selects = "involvedObject.name={PodName}&involvedObject.namespace={Namespace}&involvedObject.uid={Uid}&limit=500"
      selects = selects.replace("{PodName}", this.form.metadata.name).replace("{Namespace}", this.form.metadata.namespace).replace("{Uid}", this.form.metadata.uid)
      const res=await listEventsWithPodSelector(this.clusterName, this.namespace, selects)
      this.eventList = res.items
      
    },
    async get_pod_domain(){
      let ip = "";
      if (this.form.status.podIPs.length > 0) {
         ip = this.form.status.podIP.replaceAll(".", "-");
      }
      this.ip=ip
      if(this.form.metadata.ownerReferences && this.form.metadata.ownerReferences.length>0 ){
        const owner = this.form.metadata.ownerReferences[0];
	
        this.controller = owner.kind;
        if (this.controller == "StatefulSet") {
           const st =await getStatefulSet(this.clusterName, this.namespace,owner.name)
           if (st.spec.serviceName && st.spec.serviceName != ""){
            const headless_service=await getService(this.clusterName, st.metadata.namespace,st.spec.serviceName)
            if(headless_service.spec.clusterIP=="None"){
              this.pod_st_name = this.form.metadata.name + "." + st.spec.serviceName;
            }
           }
        }
      }
    }
  },
  async created() {
    this.clusterName = this.$route.query.cluster
    this.cluster_domain= await get_cluster_domain(this.clusterName)
    await this.getDetail()
  },
}
</script>

<style scoped>
</style>
