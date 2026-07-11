<script lang="ts" setup>
import { useToast } from 'vue-toastification'
import api from '@/api'
import type { FilterRuleGroup, Site } from '@/api/types'
import { useI18n } from 'vue-i18n'
import { useSilentSettingRefresh } from '@/composables/useSilentSettingRefresh'

// 国际化
const { t } = useI18n()

const props = defineProps({
  active: {
    type: Boolean,
    default: true,
  },
})

// 提示框
const $toast = useToast()

// 所有站点
const allSites = ref<Site[]>([])

// 选中订阅站点
const selectedSites = ref<number[]>([])

// 系统设置
const SystemSettings = ref<any>({
  Basic: {
    SEARCH_MULTIPLE_NAME: false,
    DOWNLOAD_SUBTITLE: false,
    AUTO_DOWNLOAD_USER: null,
    TORRENT_TAG: 'MOVIEPILOT',
  },
})

// 媒体信息数据源字典
const mediaSourcesDict = [
  {
    title: 'TheMovieDb',
    value: 'themoviedb',
  },
  {
    title: '豆瓣',
    value: 'douban',
  },
  {
    title: 'Bangumi',
    value: 'bangumi',
  },
]

// 当前选中的媒体信息数据源
const selectedMediaSource = ref([])

// 当前选中的过滤规则组
const selectedFilterGroup = ref([])

// 过滤规则组选择项
const filterRuleGroupOptions = computed(() => {
  return filterRuleGroups.value.map(item => ({
    title: item.name,
    value: item.name,
  }))
})

// 所有规则组列表
const filterRuleGroups = ref<FilterRuleGroup[]>([])

// 查询所有站点
async function querySites() {
  try {
    const data: any[] = await api.get('site/indexsites')
    allSites.value = data
  } catch (error) {
    console.log(error)
  }
}

// 加载规则组
async function queryFilterRuleGroups() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/UserFilterRuleGroups')
    filterRuleGroups.value = result.data?.value ?? []
  } catch (error) {
    console.log(error)
  }
}

// 查询用户选中的站点
async function querySelectedSites() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/public/IndexerSites')

    selectedSites.value = result.data?.value ?? []
  } catch (error) {
    console.log(error)
  }
}

// 保存用户选中的站点
async function saveSelectedSites() {
  try {
    // 用户名密码
    const result: { [key: string]: any } = await api.post('system/setting/IndexerSites', selectedSites.value)

    if (result.success) $toast.success('搜索站点保存成功')
    else $toast.error('搜索站点保存失败！')
  } catch (error) {
    console.log(error)
  }
}

// 调用API查询设置
async function loadSearchSetting() {
  try {
    const result1: { [key: string]: any } = await api.get('system/setting/SEARCH_SOURCE')
    if (result1.success) selectedMediaSource.value = result1.data?.value?.split(',')
    const result2: { [key: string]: any } = await api.get('system/setting/SearchFilterRuleGroups')
    if (result2.success) selectedFilterGroup.value = result2.data?.value
  } catch (error) {
    console.log(error)
  }
}

// 调用API保存设置
async function saveSystemSetting(value: { [key: string]: any }) {
  try {
    const result: { [key: string]: any } = await api.post('system/env', value)

    if (result.success) {
      return true
    }
  } catch (error) {}
  return false
}

// 调用API保存设置
async function saveSearchSetting() {
  try {
    const result1: { [key: string]: any } = await api.post(
      'system/setting/SEARCH_SOURCE',
      selectedMediaSource.value.join(','),
    )

    if (!result1 || !result1.success) {
      $toast.error(`媒体搜索数据源保存失败：${result1?.message}！`)
      return
    }

    const result2: { [key: string]: any } = await api.post(
      'system/setting/SearchFilterRuleGroups',
      selectedFilterGroup.value,
    )

    const result3 = await saveSystemSetting(SystemSettings.value.Basic)

    if (result2.success && result3) {
      $toast.success('搜索基础设置保存成功')
    } else {
      $toast.error('搜索基础设置保存失败！')
    }
  } catch (error) {
    console.log(error)
  }
}

// 加载系统设置
async function loadSystemSettings() {
  try {
    const result: { [key: string]: any } = await api.get('system/env')
    if (result.success) {
      // 将API返回的值赋值给SystemSettings
      for (const sectionKey of Object.keys(SystemSettings.value) as Array<keyof typeof SystemSettings.value>) {
        Object.keys(SystemSettings.value[sectionKey]).forEach((key: string) => {
          if (result.data.hasOwnProperty(key)) (SystemSettings.value[sectionKey] as any)[key] = result.data[key]
        })
      }
    }
  } catch (error) {
    console.log(error)
  }
}

// Jackett 配置数据
const jackettConfig = ref({
  host: '',
  api_key: '',
  password: '',
})

// 连接测试中标记
const testingJackett = ref(false)

// 查询 Jackett 配置
async function loadJackettConfig() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/Jackett')
    if (result.success && result.data?.value) {
      jackettConfig.value = {
        host: result.data.value.host ?? '',
        api_key: result.data.value.api_key ?? '',
        password: result.data.value.password ?? '',
      }
    }
  } catch (error) {
    console.log(error)
  }
}

// 保存 Jackett 配置
async function saveJackettConfig() {
  try {
    const result: { [key: string]: any } = await api.post('system/setting/Jackett', jackettConfig.value)
    if (result.success) {
      $toast.success('Jackett 配置保存成功')
      await querySites()
    } else {
      $toast.error(`Jackett 配置保存失败：${result.message}！`)
    }
  } catch (error) {
    console.log(error)
    $toast.error('Jackett 配置保存失败！')
  }
}

// 测试 Jackett 连通性
async function testJackett() {
  if (!jackettConfig.value.host || !jackettConfig.value.api_key) {
    $toast.warning('请先填写 Jackett 地址和 API Key')
    return
  }
  testingJackett.value = true
  try {
    // 自动保存当前修改值
    const saveRes: { [key: string]: any } = await api.post('system/setting/Jackett', jackettConfig.value)
    if (!saveRes.success) {
      $toast.error('保存临时配置失败，测试中断！')
      testingJackett.value = false
      return
    }
    // 测试连通性
    const result: { [key: string]: any } = await api.get('system/moduletest/jackett')
    if (result.success) {
      $toast.success(result.message || '连接测试成功')
      await querySites()
    } else {
      $toast.error(result.message || '连接测试失败，请检查配置或网络')
    }
  } catch (error) {
    console.log(error)
    $toast.error('连接测试发生异常')
  } finally {
    testingJackett.value = false
  }
}

async function loadPageData() {
  await Promise.all([querySites(), queryFilterRuleGroups(), querySelectedSites(), loadSearchSetting(), loadSystemSettings(), loadJackettConfig()])
}

onMounted(() => {
  loadPageData()
})

useSilentSettingRefresh(loadPageData, {
  active: computed(() => props.active),
})
</script>

<template>
  <VRow>
    <VCol cols="12">
      <VCard>
        <VCardItem>
          <VCardTitle>{{ t('setting.search.basicSettings') }}</VCardTitle>
          <VCardSubtitle>{{ t('setting.search.basicSettingsDesc') }}</VCardSubtitle>
        </VCardItem>
        <VCardText>
          <VRow>
            <VCol cols="12" md="6">
              <VSelect
                v-model="selectedMediaSource"
                multiple
                clearable
                chips
                :items="mediaSourcesDict"
                :label="t('setting.search.mediaSource')"
                :hint="t('setting.search.mediaSourceHint')"
                persistent-hint
                prepend-inner-icon="mdi-database-search"
              />
            </VCol>
            <VCol cols="12" md="6">
              <VAutocomplete
                v-model="selectedFilterGroup"
                multiple
                clearable
                chips
                :items="filterRuleGroupOptions"
                :label="t('setting.search.filterRuleGroup')"
                :hint="t('setting.search.filterRuleGroupHint')"
                persistent-hint
                prepend-inner-icon="mdi-filter"
              />
            </VCol>
          </VRow>
          <VRow>
            <VCol cols="12" md="6">
              <VTextField
                v-model="SystemSettings.Basic.TORRENT_TAG"
                :label="t('setting.search.downloadLabel')"
                placeholder="MOVIEPILOT"
                :hint="t('setting.search.downloadLabelHint')"
                persistent-hint
                prepend-inner-icon="mdi-tag"
              />
            </VCol>
            <VCol cols="12" md="6">
              <VCombobox
                v-model="SystemSettings.Basic.AUTO_DOWNLOAD_USER"
                :label="t('setting.search.downloadUser')"
                :placeholder="t('setting.search.downloadUserPlaceholder')"
                :hint="t('setting.search.downloadUserHint')"
                persistent-hint
                prepend-inner-icon="mdi-account"
              />
            </VCol>
            <VCol cols="12" md="6">
              <VSwitch
                v-model="SystemSettings.Basic.SEARCH_MULTIPLE_NAME"
                :label="t('setting.search.multipleNameSearch')"
                :hint="t('setting.search.multipleNameSearchHint')"
                persistent-hint
              />
            </VCol>
            <VCol cols="12" md="6">
              <VSwitch
                v-model="SystemSettings.Basic.DOWNLOAD_SUBTITLE"
                :label="t('setting.search.downloadSubtitle')"
                :hint="t('setting.search.downloadSubtitleHint')"
                persistent-hint
              />
            </VCol>
          </VRow>
        </VCardText>
        <VCardText>
          <VForm @submit.prevent="() => {}">
            <div class="d-flex flex-wrap gap-4 mt-4">
              <VBtn type="submit" @click="saveSearchSetting" prepend-icon="mdi-content-save">
                {{ t('common.save') }}
              </VBtn>
            </div>
          </VForm>
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
  <VRow>
    <VCol cols="12">
      <VCard>
        <VCardItem>
          <VCardTitle>Jackett 配置</VCardTitle>
          <VCardSubtitle>配置 Jackett 索引器以检索公网/私有 BT 资源</VCardSubtitle>
        </VCardItem>
        <VCardText>
          <VRow>
            <VCol cols="12" md="4">
              <VTextField
                v-model="jackettConfig.host"
                label="Jackett 地址"
                placeholder="http://127.0.0.1:9117"
                persistent-hint
                prepend-inner-icon="mdi-server"
              />
            </VCol>
            <VCol cols="12" md="4">
              <VTextField
                v-model="jackettConfig.api_key"
                label="API Key"
                placeholder="请输入 Jackett 的 API Key"
                persistent-hint
                prepend-inner-icon="mdi-key"
              />
            </VCol>
            <VCol cols="12" md="4">
              <VTextField
                v-model="jackettConfig.password"
                type="password"
                label="Jackett 密码"
                placeholder="若设置了访问密码请输入，否则留空"
                persistent-hint
                prepend-inner-icon="mdi-lock"
              />
            </VCol>
          </VRow>
        </VCardText>
        <VCardText>
          <VForm @submit.prevent="() => {}">
            <div class="d-flex flex-wrap gap-4 mt-4">
              <VBtn type="submit" @click="saveJackettConfig" prepend-icon="mdi-content-save">
                {{ t('common.save') }}
              </VBtn>
              <VBtn color="info" variant="outlined" :loading="testingJackett" @click="testJackett" prepend-icon="mdi-swap-horizontal">
                测试连接
              </VBtn>
            </div>
          </VForm>
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
  <VRow>
    <VCol cols="12">
      <VCard>
        <VCardItem>
          <VCardTitle>{{ t('setting.search.downloadSite') }}</VCardTitle>
          <VCardSubtitle>{{ t('setting.search.downloadSiteDesc') }}</VCardSubtitle>
        </VCardItem>
        <VCardText>
          <VChipGroup v-model="selectedSites" column multiple>
            <VChip
              v-for="site in allSites"
              :key="site.id"
              :color="selectedSites.includes(site.id) ? 'primary' : ''"
              filter
              variant="outlined"
              :value="site.id"
            >
              {{ site.name }}
            </VChip>
          </VChipGroup>
        </VCardText>
        <VCardText>
          <VForm @submit.prevent="() => {}">
            <div class="d-flex flex-wrap gap-4 mt-4">
              <VBtn type="submit" @click="saveSelectedSites" prepend-icon="mdi-content-save">
                {{ t('common.save') }}
              </VBtn>
            </div>
          </VForm>
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
</template>
