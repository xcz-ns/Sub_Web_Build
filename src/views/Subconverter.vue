<template>
  <div>
    <el-row style="margin-top: 10px">
      <el-col>
        <el-card>
          <div slot="header">
            Subscription Converter
            <svg-icon icon-class="github" style="margin-left: 20px" @click="goToProject" />

            <div style="display: inline-block; position:absolute; right: 20px">{{ backendVersion }}</div>
          </div>
          <el-container>
            <el-form :model="form" label-width="140px" label-position="left" style="width: 100%">
              <el-form-item label="模式设置:">
                <el-radio v-model="advanced" label="1">基础模式</el-radio>
                <el-radio v-model="advanced" label="2">进阶模式</el-radio>
              </el-form-item>
              <el-form-item label="订阅链接:">
                <el-input v-model="form.sourceSubUrl" type="textarea" rows="3"
                  placeholder="支持订阅或ss/ssr/vmess链接，多个链接每行一个或用 | 分隔" @blur="saveSubUrl" />
              </el-form-item>
              <el-form-item label="客户端:">
                <el-select v-model="form.clientType" style="width: 100%">
                  <el-option v-for="(v, k) in options.clientTypes" :key="k" :label="k" :value="v"></el-option>
                </el-select>
              </el-form-item>

              <div v-if="advanced === '2'">
                <el-form-item label="后端地址:">
                  <el-autocomplete style="width: 100%" v-model="form.customBackend" :fetch-suggestions="backendSearch"
                    placeholder="动动小手，（建议）自行搭建后端服务。例：http://127.0.0.1:25500/sub?">
                    <el-button slot="append" @click="gotoGayhub" icon="el-icon-link">前往项目仓库</el-button>
                  </el-autocomplete>
                </el-form-item>
                <el-form-item label="远程配置:">
                  <el-select v-model="form.remoteConfig" allow-create filterable placeholder="请选择" style="width: 100%">
                    <el-option-group v-for="group in options.remoteConfig" :key="group.label" :label="group.label">
                      <el-option v-for="item in group.options" :key="item.value" :label="item.label"
                        :value="item.value"></el-option>
                    </el-option-group>
                    <el-button slot="append" @click="gotoRemoteConfig" icon="el-icon-link">配置示例</el-button>
                  </el-select>
                </el-form-item>
                <el-form-item label="Include:">
                  <el-input v-model="form.includeRemarks" placeholder="节点名包含的关键字，支持正则" />
                </el-form-item>
                <el-form-item label="Exclude:">
                  <el-input v-model="form.excludeRemarks" placeholder="节点名不包含的关键字，支持正则" />
                </el-form-item>
                <el-form-item label="FileName:">
                  <el-input v-model="form.filename" placeholder="返回的订阅文件名" />
                </el-form-item>

                <el-form-item v-for="(param, i) in customParams" :key="i">
                  <el-input slot="label" v-model="param.name" placeholder="自定义参数名">
                    <div slot="suffix" style="width: 10px;">:</div>
                  </el-input>
                  <el-input v-model="param.value" placeholder="自定义参数内容">
                      <el-button slot="suffix" type="text" icon="el-icon-delete" style="margin-right: 5px" @click="customParams.splice(i, 1)"/>
                  </el-input>
                </el-form-item>

                <el-form-item label-width="0px">
                  <el-row type="flex">
                    <el-col>
                      <el-checkbox v-model="form.nodeList" label="输出为 Node List" border></el-checkbox>
                    </el-col>
                    <el-popover placement="bottom" v-model="form.extraset">
                      <el-row>
                        <el-checkbox v-model="form.emoji" label="Emoji"></el-checkbox>
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.scv" label="跳过证书验证"></el-checkbox>
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.udp" @change="needUdp = true" label="启用 UDP"></el-checkbox>
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.appendType" label="节点类型"></el-checkbox>
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.sort" label="排序节点"></el-checkbox>
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.fdn" label="过滤非法节点"></el-checkbox>
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.expand" label="规则展开"></el-checkbox>
                      </el-row>
                      <el-button slot="reference">更多选项</el-button>
                    </el-popover>
                    <el-popover placement="bottom" style="margin-left: 10px">
                      <el-row>
                        <el-checkbox v-model="form.tpl.surge.doh" label="Surge.DoH"></el-checkbox>
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.tpl.clash.doh" label="Clash.DoH"></el-checkbox>
                      </el-row>
                      <el-row>
                        <el-checkbox v-model="form.insert" label="网易云"></el-checkbox>
                      </el-row>
                      <el-button slot="reference">定制功能</el-button>
                    </el-popover>
                    <el-popover placement="top-end" title="添加自定义转换参数" trigger="hover">
                      <el-link type="primary" :href="subDocAdvanced" target="_blank" icon="el-icon-info">参考文档</el-link>
                      <el-button slot="reference" @click="addCustomParam" style="margin-left: 10px">
                        <i class="el-icon-plus"></i>
                      </el-button>
                    </el-popover>
                  </el-row>
                </el-form-item>
              </div>

              <div style="margin-top: 50px"></div>

              <el-divider content-position="center">
                <i class="el-icon-magic-stick"></i>
              </el-divider>

              <el-form-item label="定制订阅:">
                <el-input class="copy-content" disabled v-model="customSubUrl">
                  <el-button slot="append" v-clipboard:copy="customSubUrl" v-clipboard:success="onCopy" ref="copy-btn"
                    icon="el-icon-document-copy">复制</el-button>
                </el-input>
              </el-form-item>
              <el-form-item label="订阅短链:">
                <el-input class="copy-content" disabled v-model="curtomShortSubUrl">
                  <el-button slot="append" v-clipboard:copy="curtomShortSubUrl" v-clipboard:success="onCopy"
                    ref="copy-btn" icon="el-icon-document-copy">复制</el-button>
                </el-input>
              </el-form-item>

              <el-form-item label-width="0px" style="margin-top: 40px; text-align: center">
                <el-button style="width: 140px" type="danger" @click="makeUrl"
                  :disabled="form.sourceSubUrl.length === 0">生成订阅链接</el-button>
                <el-button style="width: 140px" type="danger" @click="makeShortUrl" :loading="loading"
                  :disabled="customSubUrl.length === 0">生成短链接</el-button>
                <!-- <el-button style="width: 140px" type="primary" @click="surgeInstall" icon="el-icon-connection">一键导入Surge</el-button> -->
              </el-form-item>

              <el-form-item label-width="0px" style="text-align: center">
                <el-button style="width: 140px" type="primary" @click="dialogUploadConfigVisible = true"
                  icon="el-icon-upload" :loading="loading">上传配置</el-button>
                <el-button style="width: 140px" type="primary" @click="clashInstall" icon="el-icon-connection"
                  :disabled="customSubUrl.length === 0">一键导入 Clash</el-button>
              </el-form-item>
              <el-form-item label-width="0px" style="text-align: center">
                <el-button style="width: 290px" type="primary" @click="dialogLoadConfigVisible = true"
                  icon="el-icon-copy-document" :loading="loading">从 URL 解析</el-button>
              </el-form-item>
            </el-form>
          </el-container>
        </el-card>
      </el-col>
    </el-row>

    <el-dialog :visible.sync="dialogUploadConfigVisible" :show-close="false" :close-on-click-modal="false"
      :close-on-press-escape="false" width="700px">
      <div slot="title">
        Remote config upload
        <el-popover trigger="hover" placement="right" style="margin-left: 10px">
          <el-link type="primary" :href="sampleConfig" target="_blank" icon="el-icon-info">参考配置</el-link>
          <i class="el-icon-question" slot="reference"></i>
        </el-popover>
      </div>
      <el-form label-position="left">
        <el-form-item prop="uploadConfig">
          <el-input v-model="uploadConfig" type="textarea" :autosize="{ minRows: 15, maxRows: 30 }" maxlength="10000"
            show-word-limit></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="uploadConfig = ''; dialogUploadConfigVisible = false">取 消</el-button>
        <el-button type="primary" @click="confirmUploadConfig" :disabled="uploadConfig.length === 0">确 定</el-button>
      </div>
    </el-dialog>

    <el-dialog :visible.sync="dialogLoadConfigVisible" :show-close="false" :close-on-click-modal="false"
      :close-on-press-escape="false" width="700px">
      <div slot="title">
        解析 Subconverter 链接
      </div>
      <el-form label-position="left" :inline="true" >
        <el-form-item prop="uploadConfig" label="订阅链接：" label-width="85px">
          <el-input v-model="loadConfig" style="width: 565px"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="loadConfig = ''; dialogLoadConfigVisible = false">取 消</el-button>
        <el-button type="primary" @click="confirmLoadConfig" :disabled="loadConfig.length === 0">确 定</el-button>
      </div>
    </el-dialog>

  </div>
</template>

<script>
const project = process.env.VUE_APP_PROJECT
const remoteConfigSample = process.env.VUE_APP_SUBCONVERTER_REMOTE_CONFIG
const subDocAdvanced = process.env.VUE_APP_SUBCONVERTER_DOC_ADVANCED
const gayhubRelease = process.env.VUE_APP_BACKEND_RELEASE
const defaultBackend = process.env.VUE_APP_SUBCONVERTER_DEFAULT_BACKEND + '/sub?'
const shortUrlBackend = process.env.VUE_APP_MYURLS_API
const configUploadBackend = process.env.VUE_APP_CONFIG_UPLOAD_API
const tgBotLink = process.env.VUE_APP_BOT_LINK

export default {
  data() {
    return {
      backendVersion: "",
      advanced: "2",

      // 是否为 PC 端
      isPC: true,

      options: {
        clientTypes: {
          Clash: "clash",
          Surge: "surge&ver=4",
          Quantumult: "quan",
          QuantumultX: "quanx",
          Mellow: "mellow",
          Surfboard: "surfboard",
          Loon: "loon",
          singbox: "singbox",
          ss: "ss",
          ssd: "ssd",
          sssub: "sssub",
          ssr: "ssr",
          ClashR: "clashr",          
          V2Ray: "v2ray",
          Trojan: "trojan",
          Surge3: "surge&ver=3",
        },
        backendOptions: [{ value: "https://clash.xczns.top:58808/sub?" }],
        remoteConfig: [
          {
            label: "Personal",
            options: [
              {
                label: "Custom_Clash",
                value:
                  "http://xczns.top:6086/VPN/Aethersailor/Custom_Clash.ini"
              },
              {
                label: "GeneralRule_Full",
                value:
                  "http://xczns.top:6086/VPN/GeneralClashRule/GeneralRule_Full.ini"
              },
              {
                label: "GeneralRule_Min",
                value:
                  "http://xczns.top:6086/VPN/GeneralClashRule/GeneralRule_Min.ini"
              },
              {
               label: "Shadowrocket",
               value:
                  "http://xczns.top:6086/VPN/GeneralClashRule/Shadowrocket.ini"
              }
            ]
          },
          {  
            label: "ACL4SSR_Local",
            options: [
              {
                label: "ACL4SSR.ini",
                value:
                   "config/ACL4SSR.ini"
              },
              {
                label: "ACL4SSR_AdblockPlus.ini",
                value:
                   "config/ACL4SSR_AdblockPlus.ini"
              },
              {
                label: "ACL4SSR_BackCN.ini",
                value:
                   "config/ACL4SSR_BackCN.ini"
              },
              {
                label: "ACL4SSR_Mini.ini",
                value:
                   "config/ACL4SSR_Mini.ini"
              },
              {
                label: "ACL4SSR_Mini_Fallback.ini",
                value:
                   "config/ACL4SSR_Mini_Fallback.ini"
              },
              {
                label: "ACL4SSR_Mini_MultiMode.ini",
                value:
                   "config/ACL4SSR_Mini_MultiMode.ini"
              },
              {
                label: "ACL4SSR_Mini_NoAuto.ini",
                value:
                   "config/ACL4SSR_Mini_NoAuto.ini"
              },
              {
                label: "ACL4SSR_NoApple.ini",
                value:
                   "config/ACL4SSR_NoApple.ini"
              },
              {
                label: "ACL4SSR_NoAuto.ini",
                value:
                   "config/ACL4SSR_NoAuto.ini"
              },
              {
                label: "ACL4SSR_NoAuto_NoApple.ini",
                value:
                   "config/ACL4SSR_NoAuto_NoApple.ini"
              },
              {
                label: "ACL4SSR_NoAuto_NoApple_NoMicrosoft.ini",
                value:
                   "config/ACL4SSR_NoAuto_NoApple_NoMicrosoft.ini"
              },
              {
                label: "ACL4SSR_NoMicrosoft.ini",
                value:
                   "config/ACL4SSR_NoMicrosoft.ini"
              },
              {
                label: "ACL4SSR_Online.ini",
                value:
                   "config/ACL4SSR_Online.ini"
              },
              {
                label: "ACL4SSR_Online_AdblockPlus.ini",
                value:
                   "config/ACL4SSR_Online_AdblockPlus.ini"
              },
              {
                label: "ACL4SSR_Online_Full.ini",
                value:
                   "config/ACL4SSR_Online_Full.ini"
              },
              {
                label: "ACL4SSR_Online_Full_AdblockPlus.ini",
                value:
                   "config/ACL4SSR_Online_Full_AdblockPlus.ini"
              },
              {
                label: "ACL4SSR_Online_Full_NoAuto.ini",
                value:
                   "config/ACL4SSR_Online_Full_NoAuto.ini"
              },
              {
                label: "ACL4SSR_Online_Mini.ini",
                value:
                   "config/ACL4SSR_Online_Mini.ini"
              },
              {
                label: "ACL4SSR_Online_Mini_AdblockPlus.ini",
                value:
                   "config/ACL4SSR_Online_Mini_AdblockPlus.ini"
              },
              {
                label: "ACL4SSR_Online_Mini_NoAuto.ini",
                value:
                   "config/ACL4SSR_Online_Mini_NoAuto.ini"
              },
              {
                label: "ACL4SSR_WithChinaIp.ini",
                value:
                   "config/ACL4SSR_WithChinaIp.ini"
              },
              {
                label: "ACL4SSR_WithChinaIp_WithGFW.ini",
                value:
                   "config/ACL4SSR_WithChinaIp_WithGFW.ini"
              },
              {
                label: "ACL4SSR_WithGFW.ini",
                value:
                   "config/ACL4SSR_WithGFW.ini"
              }
            ]
          }
        ]
      },
      form: {
        sourceSubUrl: "",
        clientType: "",
        customBackend: "https://clash.xczns.top:58808/sub?",
        remoteConfig: "",
        excludeRemarks: "官网|刷新|套餐|重置|流量|波特兰|达拉斯|俄勒冈|凤凰城|费利蒙|硅谷|拉斯维加斯|洛杉矶|圣何塞|圣克拉拉|西雅图|芝加哥|日本|川日|东京|大阪|泉日|埼玉|沪日|深日|(?<!尼|-)日|JP|Japan|🇯🇵|KR|Korea|KOR|首尔|韩|韓|加拿大|Canada|渥太华|温哥华|卡尔加里|英国|Britain|United Kingdom|England|伦敦|法国|France|巴黎|德国|Germany|柏林|法兰克福|荷兰|Netherlands|阿姆斯特丹|土耳其|Turkey|Türkiye",
        includeRemarks: "",
        filename: "红杏云",
        emoji: true,
        nodeList: false,
        extraset: false,
        sort: false,
        udp: true,
        tfo: true,
        scv: true,
        fdn: true,
        expand: true,
        appendType: false,
        insert: false, // 是否插入默认订阅的节点，对应配置项 insert_url
        new_name: true, // 是否使用 Clash 新字段

        // tpl 定制功能
        tpl: {
          surge: {
            doh: false // dns 查询是否使用 DoH
          },
          clash: {
            doh: false
          }
        }
      },

      customParams: [],

      loading: false,
      customSubUrl: "",
      curtomShortSubUrl: "",

      dialogUploadConfigVisible: false,
      loadConfig: "",
      dialogLoadConfigVisible: false,
      uploadConfig: "",
      uploadPassword: "",
      myBot: tgBotLink,
      sampleConfig: remoteConfigSample,
      subDocAdvanced: subDocAdvanced,

      needUdp: false, // 是否需要添加 udp 参数
    };
  },
  created() {
    document.title = "Subscription Converter";
    this.isPC = this.$getOS().isPc;

    // 获取 url cache
    if (process.env.VUE_APP_USE_STORAGE === 'true') {
      this.form.sourceSubUrl = this.getLocalStorageItem('sourceSubUrl')
    }
  },
  mounted() {
    this.form.clientType = "clash";
    this.notify();
    this.getBackendVersion();
  },
  methods: {
    onCopy() {
      this.$message.success("Copied!");
    },
    goToProject() {
      window.open(project);
    },
    gotoGayhub() {
      window.open(gayhubRelease);
    },
    gotoRemoteConfig() {
      window.open(remoteConfigSample);
    },
    clashInstall() {
      if (this.customSubUrl === "") {
        this.$message.error("请先填写必填项，生成订阅链接");
        return false;
      }

      const url = "clash://install-config?url=";
      window.open(
        url +
        encodeURIComponent(
          this.curtomShortSubUrl !== ""
            ? this.curtomShortSubUrl
            : this.customSubUrl
        )
      );
    },
    surgeInstall() {
      if (this.customSubUrl === "") {
        this.$message.error("请先填写必填项，生成订阅链接");
        return false;
      }

      const url = "surge://install-config?url=";
      window.open(url + this.customSubUrl);
    },
    addCustomParam(){
      this.customParams.push({
        name: "",
        value: "",
      })
    },
    makeUrl() {
      if (this.form.sourceSubUrl === "" || this.form.clientType === "") {
        this.$message.error("订阅链接与客户端为必填项");
        return false;
      }

      let backend =
        this.form.customBackend === ""
          ? defaultBackend
          : this.form.customBackend;

      let sourceSub = this.form.sourceSubUrl;
      sourceSub = sourceSub.replace(/(\n|\r|\n\r)/g, "|");

      this.customSubUrl =
        backend +
        "target=" +
        this.form.clientType +
        "&url=" +
        encodeURIComponent(sourceSub) +
        "&insert=" +
        this.form.insert;

      if (this.advanced === "2") {
        if (this.form.remoteConfig) {
          this.customSubUrl +=
            "&config=" + encodeURIComponent(this.form.remoteConfig);
        }
        if (this.form.excludeRemarks) {
          this.customSubUrl +=
            "&exclude=" + encodeURIComponent(this.form.excludeRemarks);
        }
        if (this.form.includeRemarks) {
          this.customSubUrl +=
            "&include=" + encodeURIComponent(this.form.includeRemarks);
        }
        if (this.form.filename) {
          this.customSubUrl +=
            "&filename=" + encodeURIComponent(this.form.filename);
        }
        if (this.form.appendType) {
          this.customSubUrl +=
            "&append_type=" + this.form.appendType.toString();
        }

        this.customSubUrl +=
          "&emoji=" +
          this.form.emoji.toString() +
          "&list=" +
          this.form.nodeList.toString() +
          "&tfo=" +
          this.form.tfo.toString() +
          "&scv=" +
          this.form.scv.toString() +
          "&fdn=" +
          this.form.fdn.toString() +
          "&expand=" +
          this.form.expand.toString() +
          "&sort=" +
          this.form.sort.toString();

        if (this.needUdp) {
          this.customSubUrl += "&udp=" + this.form.udp.toString()
        }

        if (this.form.tpl.surge.doh === true) {
          this.customSubUrl += "&surge.doh=true";
        }

        if (this.form.clientType === "clash") {
          if (this.form.tpl.clash.doh === true) {
            this.customSubUrl += "&clash.doh=true";
          }

          this.customSubUrl += "&new_name=" + this.form.new_name.toString();
        }

        this.customParams.filter(param => param.name && param.value).forEach(param => {
          this.customSubUrl += `&${encodeURIComponent(param.name)}=${encodeURIComponent(param.value)}`
        })
      }

      this.$copyText(this.customSubUrl);
      this.$message.success("定制订阅已复制到剪贴板");
    },
    makeShortUrl() {
      if (this.customSubUrl === "") {
        this.$message.warning("请先生成订阅链接，再获取对应短链接");
        return false;
      }

      this.loading = true;

      let data = new FormData();
      data.append("longUrl", btoa(this.customSubUrl));

      this.$axios
        .post(shortUrlBackend, data, {
          header: {
            "Content-Type": "application/form-data; charset=utf-8"
          }
        })
        .then(res => {
          if (res.data.Code === 1 && res.data.ShortUrl !== "") {
            this.curtomShortSubUrl = res.data.ShortUrl;
            this.$copyText(res.data.ShortUrl);
            this.$message.success("短链接已复制到剪贴板");
          } else {
            this.$message.error("短链接获取失败：" + res.data.Message);
          }
        })
        .catch(() => {
          this.$message.error("短链接获取失败");
        })
        .finally(() => {
          this.loading = false;
        });
    },
    notify() {
      const h = this.$createElement;

      this.$notify({
        title: "隐私提示",
        type: "warning",
        message: h(
          "i",
          { style: "color: teal" },
          "各种订阅链接（短链接服务除外）生成纯前端实现，无隐私问题。默认提供后端转换服务，隐私担忧者请自行搭建后端服务。"
        )
      });
    },
    confirmUploadConfig() {
      if (this.uploadConfig === "") {
        this.$message.warning("远程配置不能为空");
        return false;
      }

      this.loading = true;

      let body = {
        content: this.uploadConfig,
      }
      this.$axios.post(configUploadBackend, body).then(res => {
        if (res.data.code === 0 && res.data.data.url !== "") {
          this.$message.success(
            "远程配置上传成功，配置链接已复制到剪贴板，有效期三个月望知悉"
          );

          // 自动填充至『表单-远程配置』
          this.form.remoteConfig = res.data.data.url;
          this.$copyText(this.form.remoteConfig);

          this.dialogUploadConfigVisible = false;
        } else {
          this.$message.error("远程配置上传失败: " + res.data.msg);
        }
      })
        .catch(() => {
          this.$message.error("远程配置上传失败");
        })
        .finally(() => {
          this.loading = false;
        });
    },
    /**
 * Asynchronously analyzes the URL.
 *
 * @return {Promise<string>} The result of the analysis.
 */
    async analyzeUrl() {
      // Check if `loadConfig` includes "target"
      if (this.loadConfig.includes("target")) {
        // If it does, return `loadConfig`
        return this.loadConfig;
      } else {
        // Otherwise, set `loading` to true
        this.loading = true;
        try {
          // Fetch the data from `loadConfig` using GET method and follow redirects
          let response = await fetch(this.loadConfig, {
            method: "GET",
            redirect: "follow",
          });
          // Return the URL from the response
          return response.url;
        } catch (e) {
          // If an error occurs, display an error message with the error details
          this.$message.error(
            "解析短链接失败，请检查短链接服务端是否配置跨域：" + e
          );
        } finally {
          // Set `loading` to false
          this.loading = false;
        }
      }
    },
    /**
     * Confirm and load the configuration.
     *
     * @return {boolean} Returns false if the 'loadConfig' is empty, otherwise returns true.
     */
    confirmLoadConfig() {
      // Check if 'loadConfig' is empty
      if (this.loadConfig.trim() === "") {
        // Display error message if 'loadConfig' is empty
        this.$message.error("订阅链接不能为空");
        return false;
      }

      // Async function to handle the configuration loading
      (async () => {
        try {
          // Analyze the URL and extract its components
          const url = new URL(await this.analyzeUrl());

          // Set the custom backend URL
          this.form.customBackend = url.origin + url.pathname + "?";

          // Parse the URL parameters
          const params = new URLSearchParams(url.search);

          // Record parameters have been read
          const getParam = params.get.bind(params)
          const excludeParams = new Set()
          params.get = key => {
            excludeParams.add(key)
            return getParam(key)
          }

          // Get the 'target' parameter
          const target = params.get("target");

          // Set the client type based on the 'target' parameter
          if (target === "surge") {
            const ver = params.get("ver") || "4";
            this.form.clientType = target + "&ver=" + ver;
          } else {
            this.form.clientType = target;
          }

          // Set other form properties based on the URL parameters
          this.form.sourceSubUrl = params.get("url").replace(/\|/g, "\n");
          this.form.insert = params.get("insert") === "true";
          this.form.remoteConfig = params.get("config");
          this.form.excludeRemarks = params.get("exclude");
          this.form.includeRemarks = params.get("include");
          this.form.filename = params.get("filename");
          this.form.appendType = params.get("append_type") === "true";
          this.form.emoji = params.get("emoji") === "true";
          this.form.nodeList = params.get("list") === "true";
          this.form.tfo = params.get("tfo") === "true";
          this.form.scv = params.get("scv") === "true";
          this.form.fdn = params.get("fdn") === "true";
          this.form.sort = params.get("sort") === "true";
          this.form.udp = params.get("udp") === "true";
          this.form.expand = params.get("expand") === "true";
          this.form.tpl.surge.doh = params.get("surge.doh") === "true";
          this.form.tpl.clash.doh = params.get("clash.doh") === "true";
          this.form.new_name = params.get("new_name") === "true";

          // Filter custom parameters
          this.customParams = Array.from(params
            .entries()
            .filter(e => !excludeParams.has(e[0]))
            .map(e => ({ name: e[0], value: e[1] }))
          )

          // Hide the configuration dialog
          this.dialogLoadConfigVisible = false;

          // Display success message
          this.$message.success("长/短链接已成功解析为订阅信息");
        } catch (error) {
          // Display error message if URL is not valid
          this.$message.error("请输入正确的订阅地址!");
        }
      })();
    },
    backendSearch(queryString, cb) {
      let backends = this.options.backendOptions;

      let results = queryString
        ? backends.filter(this.createFilter(queryString))
        : backends;

      // 调用 callback 返回建议列表的数据
      cb(results);
    },
    createFilter(queryString) {
      return candidate => {
        return (
          candidate.value.toLowerCase().indexOf(queryString.toLowerCase()) === 0
        );
      };
    },
    getBackendVersion() {
      this.$axios
        .get(
          defaultBackend.substring(0, defaultBackend.length - 5) + "/version"
        )
        .then(res => {
          this.backendVersion = res.data.replace(/backend\n$/gm, "");
          this.backendVersion = this.backendVersion.replace("subconverter", "");
        });
    },
    saveSubUrl() {
      if (this.form.sourceSubUrl !== '') {
        this.setLocalStorageItem('sourceSubUrl', this.form.sourceSubUrl)
      }
    },
    getLocalStorageItem(itemKey) {
      const now = +new Date()
      let ls = localStorage.getItem(itemKey)

      let itemValue = ''
      if (ls !== null) {
        let data = JSON.parse(ls)
        if (data.expire > now) {
          itemValue = data.value
        } else {
          localStorage.removeItem(itemKey)
        }
      }

      return itemValue
    },
    setLocalStorageItem(itemKey, itemValue) {
      const ttl = process.env.VUE_APP_CACHE_TTL
      const now = +new Date()

      let data = {
        setTime: now,
        ttl: parseInt(ttl),
        expire: now + ttl * 1000,
        value: itemValue
      }
      localStorage.setItem(itemKey, JSON.stringify(data))
    }
  },
};
</script>
