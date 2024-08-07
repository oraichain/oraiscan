<script lang="ts" setup>
import {
  useBaseStore,
  useBlockchain,
  useFormatter,
  useMintStore,
  useStakingStore,
  useTxDialog,
} from '@/stores';
import { computed } from '@vue/reactivity';
import { onMounted, ref } from 'vue';
import { Icon } from '@iconify/vue';
import type { Key, SlashingParam } from '@/types';
import { formatSeconds } from '@/libs/utils';
import type { Validator } from 'cosmjs-types/cosmos/staking/v1beta1/staking';
import type { Params } from 'cosmjs-types/cosmos/slashing/v1beta1/slashing';
import { fromAscii, toBase64 } from '@cosmjs/encoding';
import type { Any } from 'cosmjs-types/google/protobuf/any';
import { decodeKey } from '@/libs';

const staking = useStakingStore();
const base = useBaseStore();
const format = useFormatter();
const dialog = useTxDialog();
const chainStore = useBlockchain();
const mintStore = useMintStore();

const cache = JSON.parse(localStorage.getItem('avatars') || '{}');
const avatars = ref(cache || {});
const latest = ref({} as Record<string, number>);
const yesterday = ref({} as Record<string, number>);
const tab = ref('active');
const unbondList = ref([] as Validator[]);
const slashing = ref({} as Params);

onMounted(() => {
  staking.fetchUnbondingValdiators().then((res) => {
    unbondList.value = res.concat(unbondList.value);
  });
  staking.fetchInacitveValdiators().then((res) => {
    unbondList.value = unbondList.value.concat(res);
  });
  chainStore.rpc.getSlashingParams().then((res) => {
    slashing.value = res.params;
  });
});

async function fetchChange() {
  let page = 0;

  let height = Number(base.latest?.block?.header?.height || 0);
  if (height > 14400) {
    height -= 14400;
  } else {
    height = 1;
  }
  // voting power in 24h ago
  while (page < staking.validators.length && height > 0) {
    await base.fetchValidatorByHeight(height, page).then((x) => {
      x.validators.forEach((v) => {
        yesterday.value[toBase64(v.pubkey?.data!)] = Number(v.votingPower);
      });
    });
    page += 100;
  }

  page = 0;
  // voting power for now
  while (page < staking.validators.length) {
    await base.fetchLatestValidators(page).then((x) => {
      x.validators.forEach((v) => {
        latest.value[toBase64(v.pubkey?.data!)] = Number(v.votingPower);
      });
    });
    page += 100;
  }
}

const changes = computed(() => {
  const changes = {} as Record<string, number>;
  Object.keys(latest.value).forEach((k) => {
    const l = latest.value[k] || 0;
    const y = yesterday.value[k] || 0;
    changes[k] = l - y;
  });
  return changes;
});

const change24 = (key: Key) => {
  const txt = key.key;
  // const n: number = latest.value[txt];
  // const o: number = yesterday.value[txt];
  // // console.log( txt, n, o)
  // return n > 0 && o > 0 ? n - o : 0;
  return changes.value[txt];
};

const change24Text = (value?: Any) => {
  if (!value) return '';
  const key = decodeKey(value);
  const v = change24(key);
  return v && v !== 0 ? format.showChanges(v) : '';
};

const change24Color = (value?: Any) => {
  if (!value) return '';
  const key = decodeKey(value);
  const v = change24(key);
  if (v > 0) return 'text-success';
  if (v < 0) return 'text-error';
};

const calculateRank = function (position: number) {
  let sum = 0;
  for (let i = 0; i < position; i++) {
    sum += Number(staking.validators[i]?.delegatorShares);
  }
  const percent = sum / staking.totalPower;

  switch (true) {
    case tab.value === 'active' && percent < 0.33:
      return 'error';
    case tab.value === 'active' && percent < 0.67:
      return 'warning';
    default:
      return 'primary';
  }
};

function isFeatured(
  endpoints: string[],
  who?: { website?: string; moniker: string }
) {
  if (!endpoints || !who) return false;
  return (
    endpoints.findIndex(
      (x) =>
        (who.website &&
          who.website
            ?.substring(0, who.website?.lastIndexOf('.'))
            .endsWith(x)) ||
        who?.moniker?.toLowerCase().search(x.toLowerCase()) > -1
    ) > -1
  );
}

const list = computed(() => {
  if (tab.value === 'active') {
    return staking.validators.map((x, i) => ({
      v: x,
      rank: calculateRank(i),
      logo: logo(x.description.identity),
    }));
  } else if (tab.value === 'featured') {
    const endpoint = chainStore.current?.endpoints?.rpc?.map((x) => x.provider);
    if (endpoint) {
      endpoint.push('ping');
      return staking.validators
        .filter((x) => isFeatured(endpoint, x.description))
        .map((x, i) => ({
          v: x,
          rank: 'primary',
          logo: logo(x.description.identity),
        }));
    }
    return [];
  }
  return unbondList.value.map((x, i) => ({
    v: x,
    rank: 'primary',
    logo: logo(x.description.identity),
  }));
});

const fetchAvatar = (identity: string) => {
  // fetch avatar from keybase
  return new Promise<void>((resolve) => {
    staking
      .keybase(identity)
      .then((d) => {
        if (Array.isArray(d.them) && d.them.length > 0) {
          const uri = String(d.them[0]?.pictures?.primary?.url).replace(
            'https://s3.amazonaws.com/keybase_processed_uploads/',
            ''
          );

          avatars.value[identity] = uri;
          resolve();
        } else throw new Error(`failed to fetch avatar for ${identity}`);
      })
      .catch((error) => {
        // console.error(error); // uncomment this if you want the user to see which avatars failed to load.
        resolve();
      });
  });
};

const loadAvatar = (identity: string) => {
  // fetches avatar from keybase and stores it in localStorage
  fetchAvatar(identity).then(() => {
    localStorage.setItem('avatars', JSON.stringify(avatars.value));
  });
};

const loadAvatars = () => {
  // fetches all avatars from keybase and stores it in localStorage
  const promises = staking.validators.map((validator) => {
    const identity = validator.description?.identity;

    // Here we also check whether we haven't already fetched the avatar
    if (identity && !avatars.value[identity]) {
      return fetchAvatar(identity);
    } else {
      return Promise.resolve();
    }
  });

  Promise.all(promises).then(() =>
    localStorage.setItem('avatars', JSON.stringify(avatars.value))
  );
};

const logo = (identity?: string) => {
  if (!identity || !avatars.value[identity]) return '';
  const url = avatars.value[identity] || '';
  return url.startsWith('http')
    ? url
    : `https://s3.amazonaws.com/keybase_processed_uploads/${url}`;
};

let height_in_24h = ref(0);
base.$subscribe((_, s) => {
  if (Number(s.earlest.block.header.height) !== height_in_24h.value) {
    height_in_24h.value = Number(s.earlest.block.header.height);
    fetchChange();
  }
});

loadAvatars();
</script>
<template>
  <div>
    <div class="rounded-lg grid sm:grid-cols-1 md:grid-cols-4 p-4 md:p-6 gap-4">
      <div class="box-border p-4">
        <span>
          <div
            class="relative w-9 h-9 rounded overflow-hidden flex items-center justify-center mr-2"
          >
            <!-- <Icon class="text-[#B4B7BB]" icon="mdi:trending-up" size="32" /> -->
            <img src="../../../assets/images/svg/dup-chart.svg" alt="" />
            <div
              class="absolute top-0 left-0 bottom-0 right-0 bg-[rgba(180,183,187,0.10)]"
            ></div>
          </div>
        </span>
        <span class="text-center">
          <div class="font-semibold mt-2 text-white text-[18px]">
            {{ format.percent(mintStore.inflation) }}
          </div>
          <div class="text-[14px] font-normal leading-5">
            {{ $t('staking.inflation') }}
          </div>
        </span>
      </div>
      <div class="box-border p-4">
        <span>
          <div
            class="relative w-9 h-9 rounded overflow-hidden flex items-center justify-center mr-2"
          >
            <!-- <Icon
              class="text-[#B4B7BB]"
              icon="mdi:lock-open-outline"
              size="32"
            /> -->
            <img src="../../../assets/images/svg/lock.svg" alt="" />
            <div
              class="absolute top-0 left-0 bottom-0 right-0 bg-[rgba(180,183,187,0.10)]"
            ></div>
          </div>
        </span>
        <span class="text-center">
          <div class="font-semibold mt-2 text-white text-[18px]">
            {{ formatSeconds(staking.params?.unbondingTime) }}
          </div>
          <div class="text-[14px] font-normal leading-5">
            {{ $t('staking.unbonding_time') }}
          </div>
        </span>
      </div>
      <div class="box-border p-4">
        <span>
          <div
            class="relative w-9 h-9 rounded overflow-hidden flex items-center justify-center mr-2"
          >
            <!-- <Icon
              class="text-[#B4B7BB]"
              icon="mdi:alert-octagon-outline"
              size="32"
            /> -->

            <img src="../../../assets/images/svg/warning.svg" alt="" />
            <div
              class="absolute top-0 left-0 bottom-0 right-0 bg-[rgba(180,183,187,0.10)]"
            ></div>
          </div>
        </span>
        <span class="text-center">
          <div class="font-semibold mt-2 text-white text-[18px]">
            {{ format.percent(slashing.slashFractionDoubleSign, 1e18) }}
          </div>
          <div class="text-[14px] font-normal leading-5">
            {{ $t('staking.double_sign_slashing') }}
          </div>
        </span>
      </div>
      <div class="box-border p-4">
        <span>
          <div
            class="relative w-9 h-9 rounded overflow-hidden flex items-center justify-center mr-2"
          >
            <!-- <Icon class="text-[#B4B7BB]" icon="mdi:pause" size="32" /> -->

            <img src="../../../assets/images/svg/pause.svg" alt="" />
            <div
              class="absolute top-0 left-0 bottom-0 right-0 bg-[rgba(180,183,187,0.10)]"
            ></div>
          </div>
        </span>
        <span class="text-center">
          <div class="font-semibold mt-2 text-white text-[18px]">
            {{ format.percent(slashing.slashFractionDowntime, 1e18) }}
          </div>
          <div class="text-[14px] font-normal leading-5">
            {{ $t('staking.downtime_slashing') }}
          </div>
        </span>
      </div>
    </div>

    <div
      class="pt-6 bg-[#141416] border border-[#242627] rounded-2xl mx-4 md:mx-6"
    >
      <div class="flex items-center justify-between py-1 px-2">
        <div class="customTab tabs tabs-boxed bg-transparent">
          <a
            class="tab text-gray-400"
            :class="{ 'tab-active': tab === 'featured' }"
            @click="tab = 'featured'"
            >{{ $t('staking.popular') }}</a
          >
          <a
            class="tab text-gray-400"
            :class="{ 'tab-active': tab === 'active' }"
            @click="tab = 'active'"
            >{{ $t('staking.active') }}</a
          >
          <a
            class="tab text-gray-400"
            :class="{ 'tab-active': tab === 'inactive' }"
            @click="tab = 'inactive'"
            >{{ $t('staking.inactive') }}</a
          >
        </div>

        <!-- <div class="text-lg font-semibold pr-4">
          {{ list.length }}/{{ staking.params.maxValidators }}
        </div> -->
      </div>

      <div class="px-4 pt-3 pb-4 rounded shadow">
        <div class="overflow-x-auto">
          <table class="table staking-table w-full">
            <thead>
              <tr>
                <th
                  scope="col"
                  class="uppercase"
                  style="width: 3rem; position: relative"
                >
                  {{ $t('staking.rank') }}
                </th>
                <th scope="col" class="uppercase">
                  {{ $t('staking.validator') }}
                </th>
                <th scope="col" class="text-right uppercase">
                  {{ $t('staking.voting_power') }}
                </th>
                <th scope="col" class="text-right uppercase">
                  {{ $t('staking.24h_changes') }}
                </th>
                <th scope="col" class="text-right uppercase">
                  {{ $t('staking.commission') }}
                </th>
                <th scope="col" class="text-center uppercase">
                  {{ $t('staking.actions') }}
                </th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="({ v, rank, logo }, i) in list"
                :key="v.operatorAddress"
                class="hover:bg-gray-100 dark:hover:bg-base-300"
              >
                <!-- 👉 rank -->
                <td>
                  <div
                    class="text-xs truncate relative px-2 py-1 rounded-full w-fit"
                    :class="`text-${rank}`"
                  >
                    <span
                      class="inset-x-0 inset-y-0 opacity-10 absolute"
                      :class="`bg-${rank}`"
                    ></span>
                    {{ i + 1 }}
                  </div>
                </td>
                <!-- 👉 Validator -->
                <td>
                  <div
                    class="flex items-center overflow-hidden"
                    style="max-width: 300px"
                  >
                    <div class="avatar mr-4 relative w-8 h-8 rounded-full">
                      <div
                        class="w-8 h-8 rounded-full bg-gray-400 absolute opacity-10"
                      ></div>
                      <div class="w-8 h-8 rounded-full">
                        <img
                          v-if="logo"
                          :src="logo"
                          class="object-contain"
                          @error="
                            (e) => {
                              const identity = v.description?.identity;
                              if (identity) loadAvatar(identity);
                            }
                          "
                        />
                        <Icon
                          v-else
                          class="text-3xl"
                          :icon="`mdi-help-circle-outline`"
                        />
                      </div>
                    </div>

                    <div class="flex flex-col">
                      <span
                        class="text-sm text-primary dark:text-link whitespace-nowrap overflow-hidden"
                      >
                        <RouterLink
                          :to="{
                            name: 'chain-staking-validator',
                            params: {
                              validator: v.operatorAddress,
                            },
                          }"
                          class="font-weight-medium"
                        >
                          {{ v.description?.moniker }}
                        </RouterLink>
                      </span>
                      <span class="text-xs">{{
                        v.description?.website || v.description?.identity || '-'
                      }}</span>
                    </div>
                  </div>
                </td>

                <!-- 👉 Voting Power -->
                <td class="text-right">
                  <div class="flex flex-col">
                    <h6 class="text-sm font-weight-medium whitespace-nowrap">
                      {{
                        format.formatToken(
                          {
                            amount: parseInt(v.tokens).toString(),
                            denom: staking.params.bondDenom,
                          },
                          true,
                          '0,0'
                        )
                      }}
                    </h6>
                    <span class="text-xs">{{
                      format.calculatePercent(
                        v.delegatorShares,
                        staking.totalPower
                      )
                    }}</span>
                  </div>
                </td>
                <!-- 👉 24h Changes -->
                <td
                  class="text-right text-xs"
                  :class="change24Color(v.consensusPubkey)"
                >
                  {{ change24Text(v.consensusPubkey) }}
                </td>
                <!-- 👉 commission -->
                <td class="text-right text-xs">
                  {{
                    format.formatCommissionRate(
                      v.commission?.commissionRates?.rate,
                      1e18
                    )
                  }}
                </td>
                <!-- 👉 Action -->
                <td class="text-center">
                  <div class="flex items-center justify-center">
                    <div
                      v-if="v.jailed"
                      class="badge badge-error gap-2 text-white"
                    >
                      {{ $t('staking.jailed') }}
                    </div>
                    <label
                      v-else
                      for="delegate"
                      class="btn-third !py-1 !px-3 capitalize !h-[unset]"
                      @click="
                        dialog.open('delegate', {
                          validator_address: v.operatorAddress,
                        })
                      "
                      >{{ $t('account.btn_delegate') }}</label
                    >
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="divider"></div>
        <div class="flex flex-row items-center">
          <div
            class="text-xs truncate relative py-2 px-4 rounded-md w-fit text-error mr-2"
          >
            <span
              class="inset-x-0 inset-y-0 opacity-10 absolute bg-error"
            ></span>
            {{ $t('staking.top') }} 33%
          </div>
          <div
            class="text-xs truncate relative py-2 px-4 rounded-md w-fit text-warning"
          >
            <span
              class="inset-x-0 inset-y-0 opacity-10 absolute bg-warning"
            ></span>
            {{ $t('staking.top') }} 67%
          </div>
          <div class="text-xs hidden md:!block pl-2">
            {{ $t('staking.description') }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<route>
  {
    meta: {
      i18n: 'staking',
      order: 3
    }
  }
</route>
