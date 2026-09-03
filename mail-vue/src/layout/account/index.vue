<template>
  <div class="account-box">
    <div class="head-opt">
      <Icon v-perm="'account:add'" class="icon add" icon="ion:add-outline" width="22" height="22" title="添加邮箱" @click="add"/>
      <Icon class="icon refresh" icon="ion:reload" width="18" height="18" title="刷新" @click="refresh"/>
      <Icon
        class="icon expand-toggle"
        :icon="isAllExpanded ? 'fluent:collapse-all-20-filled' : 'fluent:expand-all-20-filled'"
        width="18"
        height="18"
        :title="isAllExpanded ? '全部折叠' : '全部展开'"
        @click="toggleExpandAll"
      />
      <div class="search-wrap">
        <el-input
          v-model="searchKeyword"
          placeholder="搜索..."
          clearable
          size="small"
          class="search-input"
        >
          <template #prefix>
            <Icon icon="ion:search-outline" width="14" height="14"/>
          </template>
        </el-input>
      </div>
    </div>
    <el-scrollbar class="scrollbar" ref="scrollbarRef">
      <div v-infinite-scroll="getAccountList" :infinite-scroll-distance="600" :infinite-scroll-immediate="false">
        <el-collapse v-model="activeDomains" class="domain-collapse">
          <el-collapse-item
            v-for="group in domainGroups"
            :key="group.domain"
            :name="group.domain"
            class="domain-group-item"
          >
            <template #title>
              <div class="domain-header">
                <div class="domain-title-wrap">
                  <Icon icon="fluent:folder-20-filled" class="domain-folder-icon" width="16" height="16" color="#409EFF"/>
                  <span class="domain-name">{{ group.domain }}</span>
                  <span class="domain-count">({{ group.accounts.length }})</span>
                </div>
                <el-badge
                  v-if="group.unreadCount > 0"
                  :value="group.unreadCount"
                  class="domain-unread-badge"
                  type="danger"
                />
              </div>
            </template>

            <div
              v-for="(item, index) in group.accounts"
              :key="item.accountId"
              class="item"
              :class="itemBg(item.accountId)"
              @click="changeAccount(item)"
            >
              <div class="account-row-content">
                <span class="account-text" :title="item.email">{{ item.email }}</span>
                <div class="account-right-actions">
                  <el-badge
                    v-if="item.unreadCount > 0"
                    :value="item.unreadCount"
                    class="account-unread-badge"
                    type="danger"
                  />
                  <div class="settings-wrap" @click.stop>
                    <el-dropdown trigger="hover">
                      <Icon icon="fluent:settings-24-filled" width="18" height="18" class="gear-icon" color="#909399"/>
                      <template #dropdown>
                        <el-dropdown-menu>
                          <el-dropdown-item @click="copyAccount(item.email)">
                            <Icon icon="fluent-color:clipboard-24" width="16" height="16" style="margin-right: 6px"/>
                            复制邮箱地址
                          </el-dropdown-item>
                          <el-dropdown-item @click="setAllReceive(item)">
                            <Icon :icon="item.allReceive ? 'flat-color-icons:folder' : 'eva:email-fill'" width="16" height="16" style="margin-right: 6px"/>
                            {{ item.allReceive ? '关闭聚合接收' : '开启聚合接收' }}
                          </el-dropdown-item>
                          <el-dropdown-item v-if="hasPerm('email:send')" @click="openSetName(item)">
                            <Icon icon="fluent:rename-16-regular" width="16" height="16" style="margin-right: 6px"/>
                            {{ $t('rename') }}
                          </el-dropdown-item>
                          <el-dropdown-item v-if="item.accountId !== userStore.user.account.accountId" @click="setAsTop(item, index)">
                            <Icon icon="fluent:pin-16-regular" width="16" height="16" style="margin-right: 6px"/>
                            {{ $t('pin') }}
                          </el-dropdown-item>
                          <el-dropdown-item v-if="item.accountId !== userStore.user.account.accountId && hasPerm('account:delete')" @click="remove(item)">
                            <Icon icon="fluent:delete-16-regular" width="16" height="16" style="margin-right: 6px"/>
                            {{ $t('delete') }}
                          </el-dropdown-item>
                        </el-dropdown-menu>
                      </template>
                    </el-dropdown>
                  </div>
                </div>
              </div>
            </div>
          </el-collapse-item>
        </el-collapse>

        <!-- Initial Loading Skeleton -->
        <template v-if="loading">
          <el-skeleton v-for="i in skeletonRows" :key="i" animated>
            <template #template>
              <el-card class="item">
                <el-skeleton-item variant="p" style="width: 70%; height: 20px; margin-bottom: 25px"/>
                <div style="display: flex; justify-content: space-between">
                  <el-skeleton-item variant="text" style="width: 20px"/>
                  <el-skeleton-item variant="text" style="width: 20px"/>
                </div>
              </el-card>
            </template>
          </el-skeleton>
        </template>

        <!-- Follow Loading Skeleton -->
        <template v-if="accounts.length > 0 && !noLoading">
          <el-skeleton animated>
            <template #template>
              <el-card class="item">
                <el-skeleton-item variant="p" style="width: 70%; height: 20px; margin-bottom: 20px"/>
                <div style="display: flex; justify-content: space-between">
                  <el-skeleton-item variant="text" style="width: 20px"/>
                  <el-skeleton-item variant="text" style="width: 20px"/>
                </div>
              </el-card>
            </template>
          </el-skeleton>
        </template>

        <div class="noLoading" v-if="noLoading && accounts.length > 0">
          <div>{{ $t('noMoreData') }}</div>
        </div>
        <div class="empty" v-if="noLoading && accounts.length === 0">
          <el-empty :description="$t('noMessagesFound')"/>
        </div>
      </div>

    </el-scrollbar>
    <el-dialog v-model="showAdd" :title="$t('addAccount')">
      <div class="container">
        <el-input v-model="addForm.email" ref="addRef" type="text" :placeholder="$t('emailAccount')" autocomplete="off" @keyup.enter="submit">
          <template #append>
            <div @click.stop="openSelect">
              <el-select
                  ref="mySelect"
                  v-model="addForm.suffix"
                  :placeholder="$t('select')"
                  class="select"
              >
                <el-option
                    v-for="item in domainList"
                    :key="item"
                    :label="item"
                    :value="item"
                />
              </el-select>
              <div>
                <span>{{ addForm.suffix }}</span>
                <Icon class="setting-icon" icon="mingcute:down-small-fill" width="20" height="20"/>
              </div>
            </div>
          </template>
        </el-input>
        <el-button class="btn" type="primary" @click="submit" :loading="addLoading"
        >{{ $t('add') }}
        </el-button>
      </div>
      <div
          class="add-email-turnstile"
          :class="verifyShow ? 'turnstile-show' : 'turnstile-hide'"
          :data-sitekey="settingStore.settings.siteKey"
          data-callback="onTurnstileSuccess"
          data-error-callback="onTurnstileError"
      >
        <span style="font-size: 12px;color: #F56C6C" v-if="botJsError">{{ $t('verifyModuleFailed') }}</span>
      </div>
    </el-dialog>
    <el-dialog v-model="setNameShow" :title="$t('changeUserName')">
      <div class="container">
        <el-input v-model="accountName" type="text" :placeholder="$t('username')" autocomplete="off" @keyup.enter="setName">
        </el-input>
        <el-button class="btn" type="primary" @click="setName" :loading="setNameLoading"
        >{{ $t('save') }}
        </el-button>
      </div>
    </el-dialog>
  </div>
</template>
<script setup>
import {Icon} from "@iconify/vue";
import {computed, nextTick, reactive, ref, watch} from "vue";
import {
  accountList,
  accountAdd,
  accountDelete,
  accountSetName,
  accountSetAllReceive,
  accountSetAsTop
} from "@/request/account.js";
import {sleep} from "@/utils/time-utils.js"
import {isEmail} from "@/utils/verify-utils.js";
import {useSettingStore} from "@/store/setting.js";
import {useAccountStore} from "@/store/account.js";
import {useEmailStore} from "@/store/email.js";
import {useUserStore} from "@/store/user.js";
import {hasPerm} from "@/perm/perm.js"
import {useI18n} from "vue-i18n";
import {AccountAllReceiveEnum} from "@/enums/account-enum.js";

const {t} = useI18n();
const userStore = useUserStore();
const accountStore = useAccountStore();
const settingStore = useSettingStore();
const emailStore = useEmailStore();
const showAdd = ref(false)
const addLoading = ref(false);
const domainList = computed(() => settingStore.domainList)
const accounts = reactive([])
const searchKeyword = ref('')
const activeDomains = ref([])
const isAllExpanded = ref(true)
const noLoading = ref(false)
const loading = ref(false)
const followLoading = ref(false);
const verifyShow = ref(false)
const setNameShow = ref(false)
const setNameLoading = ref(false)
const accountName = ref(null)
const addRef = ref({})
const scrollbarRef = ref({})
let account = null
let turnstileId = null
const botJsError = ref(false)
let verifyToken = ''
let verifyErrorCount = 0
let first = true
const addForm = reactive({
  email: '',
  suffix: settingStore.domainList[0]
})
let skeletonRows = 10
const queryParams = {
  size: 30
}

const mySelect = ref()

const domainGroups = computed(() => {
  const kw = searchKeyword.value.trim().toLowerCase();
  const map = {};

  accounts.forEach(acc => {
    const emailStr = acc.email || '';
    const parts = emailStr.split('@');
    const domain = parts[1] || '其他';

    if (kw) {
      if (!domain.toLowerCase().includes(kw) && !emailStr.toLowerCase().includes(kw)) {
        return;
      }
    }

    if (!map[domain]) {
      map[domain] = {
        domain,
        unreadCount: 0,
        accounts: []
      };
    }
    map[domain].accounts.push(acc);
    map[domain].unreadCount += (acc.unreadCount || 0);
  });

  return Object.values(map);
});

watch(domainGroups, (groups) => {
  if (groups.length > 0 && activeDomains.value.length === 0 && isAllExpanded.value) {
    activeDomains.value = groups.map(g => g.domain);
  }
}, { immediate: true });

function toggleExpandAll() {
  if (isAllExpanded.value) {
    activeDomains.value = [];
    isAllExpanded.value = false;
  } else {
    activeDomains.value = domainGroups.value.map(g => g.domain);
    isAllExpanded.value = true;
  }
}

if (hasPerm('account:query')) {
  getAccountList()
}

watch(() => accountStore.changeUserAccountName, () => {
  accounts[0].name = accountStore.changeUserAccountName
})

watch(() => settingStore.domainList, (list) => {
  if (!addForm.suffix && list.length > 0) {
    addForm.suffix = list[0]
  }
}, {immediate: true})


const openSelect = () => {
  mySelect.value.toggleMenu()
}

window.onTurnstileError = (e) => {
  if (verifyErrorCount >= 4) {
    return
  }
  verifyErrorCount++
  console.warn('人机验加载失败', e)
  setTimeout(() => {
    nextTick(() => {
      if (!turnstileId) {
        turnstileId = window.turnstile.render('.add-email-turnstile')
      } else {
        window.turnstile.reset(turnstileId);
      }
    })
  }, 1500)
};

window.onTurnstileSuccess = (token) => {
  verifyToken = token;
};

function getSkeletonRows() {
  if (accounts.length > 20) return skeletonRows = 20
  if (accounts.length === 0) return skeletonRows = 1
  skeletonRows = accounts.length
}

function setName() {

  if (setNameLoading.value) return

  let name = accountName.value

  if (name === account.name) {
    setNameShow.value = false
    return
  }

  if (!name) {
    ElMessage({
      message: t('emptyUserNameMsg'),
      type: 'error',
      plain: true,
    })
    return;
  }

  setNameLoading.value = true
  accountSetName(account.accountId, name).then(() => {
    account.name = name
    setNameShow.value = false

    if (account.accountId === userStore.user.account.accountId) {
      userStore.user.name = name
    }

    ElMessage({
      message: t('saveSuccessMsg'),
      type: "success",
      plain: true
    })
  }).finally(() => {
    setNameLoading.value = false
  })
}

function openSetName(accountItem) {
  accountName.value = accountItem.name
  account = accountItem
  setNameShow.value = true
}

function setAllReceive(account) {
  let allReceiveAccount = accounts.find(account => account.allReceive === AccountAllReceiveEnum.ENABLED);
  if (allReceiveAccount && allReceiveAccount.accountId !== account.accountId) allReceiveAccount.allReceive = AccountAllReceiveEnum.DISABLED;
  account.allReceive = account.allReceive === AccountAllReceiveEnum.DISABLED ? AccountAllReceiveEnum.ENABLED : AccountAllReceiveEnum.DISABLED;
  accountSetAllReceive(account.accountId).catch(() => {
    account.allReceive = account.allReceive === AccountAllReceiveEnum.DISABLED ? AccountAllReceiveEnum.ENABLED : AccountAllReceiveEnum.DISABLED;
    if (allReceiveAccount) allReceiveAccount.allReceive = AccountAllReceiveEnum.ENABLED;
  }).then(() => {
    if (account.allReceive === AccountAllReceiveEnum.ENABLED) {
      ElMessage({
        message: t('setSuccess'),
        type: 'success',
        plain: true,
      })
    }
    changeAccount(account);
    emailStore.emailScroll?.refreshList();
    emailStore.sendScroll?.refreshList();
  })
}


function showNullSetting(item) {
  return !hasPerm('email:send') && !(item.accountId !== userStore.user.account.accountId && hasPerm('account:delete'))
}

function itemBg(accountId) {
  return accountStore.currentAccountId === accountId ? 'item-choose' : ''
}



function remove(account) {
  ElMessageBox.confirm(t('delConfirm', {msg: account.email}), {
    confirmButtonText: t('confirm'),
    cancelButtonText: t('cancel'),
    type: 'warning'
  }).then(() => {
    accountDelete(account.accountId).then(() => {
      const index = accounts.findIndex(item => item.accountId === account.accountId);
      accounts.splice(index, 1);
      if (accounts.length < queryParams.size) {
        getAccountList()
      }
      ElMessage({
        message: t('delSuccessMsg'),
        type: 'success',
        plain: true,
      })
    })
  });
}

function refresh() {
  if (loading.value) {
    return
  }
  loading.value = false
  followLoading.value = false
  noLoading.value = false
  queryParams.accountId = 0
  queryParams.lastSort = null
  getSkeletonRows();
  scrollbarRef.value.setScrollTop(0)
  accounts.splice(0, accounts.length)
  getAccountList()
}

function changeAccount(account) {
  accountStore.currentAccountId = account.accountId
  accountStore.currentAccount = account
}

function add() {
  addForm.suffix = addForm.suffix || settingStore.domainList[0]
  showAdd.value = true
  setTimeout(() => {
    addRef.value.focus()
  }, 100)
}

function setAsTop(account, index) {
  accountSetAsTop(account.accountId).then(() => {
    ElMessage({
      message: t('setSuccess'),
      type: 'success',
      plain: true,
    })

    const [item] = accounts.splice(index, 1);
    accounts.splice(1, 0, item);

  });
}

async function copyAccount(account) {
  try {
    await navigator.clipboard.writeText(account);
    ElMessage({
      message: t('copySuccessMsg'),
      type: 'success',
      plain: true,
    })
  } catch (err) {
    console.error(`${t('copyFailMsg')}:`, err);
    ElMessage({
      message: t('copyFailMsg'),
      type: 'error',
      plain: true,
    })
  }
}

function getAccountList() {

  if (loading.value || followLoading.value || noLoading.value) return;

  if (accounts.length === 0) {
    loading.value = true
  } else {
    followLoading.value = true
  }

  let start = Date.now();

  const accountId = accounts.length > 0 ? accounts.at(-1).accountId : 0;
  const lastSort = accounts.length > 0 ? accounts.at(-1).sort : null;

  accountList(accountId, queryParams.size, lastSort).then(async list => {

    let end = Date.now();
    let duration = end - start;
    if (duration < 300) {
      await sleep(300 - duration)
    }

    if (list.length < queryParams.size) {
      noLoading.value = true
    }
    if (accounts.length === 0) {
      accountStore.currentAccount = list[0]
    }

    accounts.push(...list)

    loading.value = false
    followLoading.value = false
    first = false
  }).catch(() => {
    loading.value = false
    followLoading.value = false
  })
}


function submit() {

  if (addLoading.value) return

  if (!addForm.email) {
    ElMessage({
      message: t('emptyEmailMsg'),
      type: "error",
      plain: true
    })
    return
  }

  if (addForm.email.length < settingStore.settings.minEmailPrefix) {
    ElMessage({
      message: t('minEmailPrefix', {msg: settingStore.settings.minEmailPrefix}),
      type: 'error',
      plain: true,
    })
    return
  }

  if (!isEmail(addForm.email + addForm.suffix)) {
    ElMessage({
      message: t('notEmailMsg'),
      type: "error",
      plain: true
    })
    return
  }

  if (!verifyToken && (settingStore.settings.addEmailVerify === 0 || (settingStore.settings.addEmailVerify === 2 && settingStore.settings.addVerifyOpen))) {
    if (!verifyShow.value) {
      verifyShow.value = true
      nextTick(() => {
        if (!turnstileId) {
          try {
            turnstileId = window.turnstile.render('.add-email-turnstile')
          } catch (e) {
            botJsError.value = true
            console.log('人机验证js加载失败')
          }
        } else {
          window.turnstile.reset('.add-email-turnstile')
        }
      })
    } else if (!botJsError.value) {
      ElMessage({
        message: t('botVerifyMsg'),
        type: "error",
        plain: true
      })
    }
    return;
  }

  addLoading.value = true
  accountAdd(addForm.email + addForm.suffix, verifyToken).then(account => {
    addLoading.value = false
    addForm.email = ''
    accounts.push(account)
    verifyToken = ''
    settingStore.settings.addVerifyOpen = account.addVerifyOpen
    ElMessage({
      message: t('addSuccessMsg'),
      type: "success",
      plain: true
    })
    verifyShow.value = false
    showAdd.value = false
    userStore.refreshUserInfo()
  }).catch(res => {
    if (res.code === 400) {
      verifyToken = ''
      if (turnstileId) {
        window.turnstile.reset(turnstileId)
      } else {
        nextTick(() => {
          turnstileId = window.turnstile.render('.add-email-turnstile')
        })
      }
      verifyShow.value = true
    }
    addLoading.value = false
  })
}
</script>
<style>
path[fill="#ffdda1"] {
  fill: #ffdd7d;
}
</style>
<style scoped lang="scss">
.account-box {

  border-right: 1px solid var(--el-border-color) !important;
  background-color: var(--el-bg-color);
  height: 100%;
  overflow: hidden;

  .head-opt {
    display: flex;
    align-items: center;
    height: 42px;
    box-shadow: var(--header-actions-border);
    padding: 0 8px;
    gap: 6px;

    .icon {
      cursor: pointer;
      flex-shrink: 0;
      color: var(--el-text-color-regular);
      &:hover {
        color: #409EFF;
      }
    }

    .search-wrap {
      flex: 1;
      .search-input {
        width: 100%;
      }
    }
  }

  .domain-collapse {
    border: none;
    :deep(.el-collapse-item__header) {
      background-color: transparent;
      padding: 0 10px;
      font-size: 13px;
      font-weight: 600;
      height: 38px;
      line-height: 38px;
      border-bottom: 1px solid var(--el-border-color-lighter);
    }
    :deep(.el-collapse-item__wrap) {
      background-color: transparent;
      border-bottom: none;
    }
    :deep(.el-collapse-item__content) {
      padding-bottom: 4px;
    }
  }

  .domain-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    width: 100%;
    padding-right: 8px;

    .domain-title-wrap {
      display: flex;
      align-items: center;
      gap: 6px;
      overflow: hidden;
      white-space: nowrap;

      .domain-name {
        font-weight: 600;
        font-size: 13px;
        color: var(--el-text-color-primary);
      }

      .domain-count {
        font-size: 11px;
        color: var(--el-text-color-secondary);
      }
    }

    .domain-unread-badge {
      :deep(.el-badge__content) {
        font-size: 10px;
        height: 16px;
        line-height: 16px;
        padding: 0 5px;
      }
    }
  }

  .scrollbar {
    width: 100%;
    height: calc(100% - 42px);
    overflow: auto;
    @media (max-width: 767px) {
      height: calc(100% - 98px);
    }

    .empty {
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100%;
    }

    .noLoading {
      display: flex;
      justify-content: center;
      align-items: center;
      padding: 10px 0;
      color: var(--secondary-text-color);
    }
  }

  .btn {
    width: 100%;
    margin-top: 15px;
  }

  .item {
    background-color: var(--el-bg-color);
    border-radius: 6px;
    padding: 8px 10px;
    margin-top: 4px;
    margin-left: 6px;
    margin-right: 6px;
    cursor: pointer;
    border: 1px solid var(--el-border-color-extra-light);

    &:hover {
      background-color: var(--el-fill-color-light);

      .gear-icon {
        opacity: 1 !important;
      }
    }

    .account-row-content {
      display: flex;
      align-items: center;
      justify-content: space-between;
      width: 100%;
      height: 24px;

      .account-text {
        font-weight: 500;
        font-size: 12px;
        color: var(--el-text-color-primary);
        overflow: hidden;
        white-space: nowrap;
        text-overflow: ellipsis;
        flex: 1;
        margin-right: 6px;
      }

      .account-right-actions {
        display: flex;
        align-items: center;
        gap: 6px;
        flex-shrink: 0;

        .account-unread-badge {
          :deep(.el-badge__content) {
            font-size: 10px;
            height: 15px;
            line-height: 15px;
            padding: 0 4px;
          }
        }

        .settings-wrap {
          display: flex;
          align-items: center;

          .gear-icon {
            opacity: 0.4;
            transition: opacity 0.2s ease;
            cursor: pointer;

            &:hover {
              color: #409EFF !important;
            }
          }
        }
      }
    }
  }

  .item-choose {
    background: var(--choose-account-background) !important;
    border-color: #409eff44;
  }
}


.setting-icon {
  position: relative;
  top: 6px;
}

:deep(.el-input-group__append) {
  padding: 0 !important;
  padding-left: 8px !important;
  background: var(--el-bg-color);
}

:deep(.el-dialog) {
  width: 400px !important;
  @media (max-width: 440px) {
    width: calc(100% - 40px) !important;
    margin-right: 20px !important;
    margin-left: 20px !important;
  }
}

.select {
  position: absolute;
  right: 30px;
  width: 100px;
  opacity: 0;
  pointer-events: none;
}

:deep(.el-pagination .el-select) {
  width: 100px;
  background: var(--el-bg-color);
}

.add-email-turnstile {
  margin-top: 15px;
}

.turnstile-show {
  opacity: 1;
}

.turnstile-hide {
  opacity: 0;
  pointer-events: none;
  position: fixed;
}

</style>
