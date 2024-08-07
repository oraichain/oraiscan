<script lang="ts" setup>
import PaginationBar from '@/components/PaginationBar.vue';
import type { ExtraTxSearchResponse } from '@/libs/client';
import { formatSeconds } from '@/libs/utils';
import { useBaseStore, useBlockchain, useFormatter } from '@/stores';
import { PageRequest } from '@/types';
import { toHex } from '@cosmjs/encoding';
import { Icon } from '@iconify/vue';
import type { IdentifiedChannel } from 'cosmjs-types/ibc/core/channel/v1/channel';
import { State } from 'cosmjs-types/ibc/core/channel/v1/channel';
import type { IdentifiedClientState } from 'cosmjs-types/ibc/core/client/v1/client';
import type { ConnectionEnd } from 'cosmjs-types/ibc/core/connection/v1/connection';
import { ClientState as TendermintClientState } from 'cosmjs-types/ibc/lightclients/tendermint/v1/tendermint';
import { computed, onMounted, ref } from 'vue';
import { useIBCModule } from '../connStore';

const props = defineProps(['chain', 'connection_id']);
const chainStore = useBlockchain();
const baseStore = useBaseStore();
const format = useFormatter();
const ibcStore = useIBCModule();
const conn = ref({} as ConnectionEnd | undefined);
const clientState = ref(
  {} as (IdentifiedClientState & TendermintClientState) | undefined
);
const channels = ref([] as IdentifiedChannel[]);

const connId = computed(() => {
  return props.connection_id || 0;
});

const loading = ref(false);
const txs = ref({} as ExtraTxSearchResponse);
const direction = ref('');
const channel_id = ref('');
const port_id = ref('');
const page = ref(new PageRequest());
page.value.limit = 5;

onMounted(() => {
  if (connId.value) {
    chainStore.rpc.getIBCConnectionsById(connId.value).then((x) => {
      conn.value = x.connection;
    });
    chainStore.rpc.getIBCConnectionsClientState(connId.value).then((x) => {
      // @ts-ignore
      clientState.value = x.identifiedClientState;
      if (x.identifiedClientState?.clientState) {
        Object.assign(
          clientState.value!,
          TendermintClientState.decode(
            x.identifiedClientState?.clientState.value
          )
        );
      }
    });
    chainStore.rpc.getIBCConnectionsChannels(connId.value).then((x) => {
      channels.value = x.channels;
    });
  }
});

function loadChannel(channel: string, port: string) {
  chainStore.rpc.getIBCChannelNextSequence(channel, port).then((x) => {
    console.log(x);
  });
}

function pageload(pageNum: number) {
  if (direction.value === 'In') {
    fetchSendingTxs(channel_id.value, port_id.value, pageNum - 1);
  } else {
    fetchSendingTxs(channel_id.value, port_id.value, pageNum - 1);
  }
}

function fetchSendingTxs(channel: string, port: string, pageNum = 0) {
  page.value.setPage(pageNum);
  loading.value = true;
  direction.value = 'Out';
  channel_id.value = channel;
  port_id.value = port;
  txs.value = {} as ExtraTxSearchResponse;
  chainStore.rpc
    .getTxs(
      [
        {
          key: 'send_packet.packet_src_channel',
          value: channel,
        },
        {
          key: 'send_packet.packet_src_port',
          value: port,
        },
      ],
      page.value
    )
    .then((res) => {
      txs.value = res;
    })
    .finally(() => (loading.value = false));
}
function fetchRecevingTxs(channel: string, port: string, pageNum = 0) {
  page.value.setPage(pageNum);
  loading.value = true;
  direction.value = 'In';
  channel_id.value = channel;
  port_id.value = port;
  txs.value = {} as ExtraTxSearchResponse;
  chainStore.rpc
    .getTxs(
      [
        {
          key: 'recv_packet.packet_dst_channel',
          value: channel,
        },
        {
          key: 'recv_packet.packet_dst_port',
          value: port,
        },
      ],
      page.value
    )
    .then((res) => {
      txs.value = res;
    })
    .finally(() => (loading.value = false));
}

function color(v: string) {
  if (v && v.indexOf('_OPEN') > -1) {
    return 'success';
  }
  return 'warning';
}
</script>
<template>
  <div class="mx-6">
    <div class="grid grid-cols-1 md:grid-cols-5 gap-4 mb-4">
      <div
        class="col-span-1 md:col-span-2 px-4 pt-3 pb-4 rounded-[16px] shadow bg-[#141416] border border-[#242627] flex flex-col justify-center items-center bg-img-ibc"
      >
        <div
          class="box-border p-4 md:p-6 min-h-[112px] min-w-[256px] !rounded-[20px]"
        >
          <div>
            <div
              class="order-first text-3xl font-semibold text-center tracking-tight text-white mb-1"
            >
              {{ baseStore.latest?.block?.header?.chainId }}
            </div>
            <div
              class="text-sm text-gray-500 dark:text-gray-400 text-center mt-4"
            >
              {{ conn?.clientId }} {{ props.connection_id }}
            </div>
          </div>
        </div>
        <div class="flex w-full gap-2 items-center justify-between my-4">
          <div class="flex-1 border border-[#383B40]"></div>
          <div
            :class="{ 'text-[#39DD47]': conn?.state === State.STATE_OPEN }"
            class="inline-flex gap-2 whitespace-nowrap"
          >
            <!-- <span class="text-lg rounded-full">&#x21cc;</span> -->
            <Icon icon="mdi:swap-vertical" width="20" height="20" />
            <div class="text-center">
              {{ conn?.state !== State.STATE_OPEN ? 'Failed' : 'Open' }}
              ({{ conn?.state }})
            </div>
          </div>
          <div class="flex-1 border border-[#383B40]"></div>
        </div>
        <div
          class="box-border p-4 md:p-6 min-h-[112px] min-w-[256px] !rounded-[20px]"
        >
          <div
            class="order-first text-3xl font-semibold tracking-tight text-center text-white"
          >
            {{ clientState?.chainId }}
          </div>
          <div
            class="text-sm text-gray-500 dark:text-gray-400 text-center mt-4"
          >
            {{ conn?.counterparty?.connectionId }}
            {{ clientState?.clientId }}
          </div>
        </div>
      </div>

      <div class="section col-span-1 md:col-span-3 !mb-0">
        <h2 class="card-title overflow-hidden text-white ml-2">
          {{ $t('ibc.title_2') }}
        </h2>
        <span class="ml-2 text-sm text-break text-[#B4B7BB] mb-2">{{
          clientState?.clientState?.typeUrl
        }}</span>
        <br />
        <br />
        <div class="w-full border-b border-base-300 mb-2"></div>
        <div class="overflow-x-auto grid grid-cols-1 md:grid-cols-2 gap-4">
          <table class="table table-sm capitalize">
            <thead class="text-white font-semibold">
              <tr>
                <td colspan="3">{{ $t('ibc.trust_parameters') }}</td>
              </tr>
            </thead>
            <tbody>
              <tr>
                <td class="w-52">{{ $t('ibc.trust_level') }}:</td>
                <td>
                  {{ clientState?.trustLevel?.numerator }}/{{
                    clientState?.trustLevel?.denominator
                  }}
                </td>
              </tr>
              <tr>
                <td class="w-52">{{ $t('ibc.trusting_period') }}:</td>
                <td>
                  {{ formatSeconds(clientState?.trustingPeriod) }}
                </td>
              </tr>
              <tr>
                <td class="w-52">{{ $t('ibc.unbonding_period') }}:</td>
                <td>
                  {{ formatSeconds(clientState?.unbondingPeriod) }}
                </td>
              </tr>
              <tr>
                <td class="w-52">{{ $t('ibc.max_clock_drift') }}:</td>
                <td>
                  {{ formatSeconds(clientState?.maxClockDrift) }}
                </td>
              </tr>
              <tr>
                <td class="w-52">{{ $t('ibc.frozen_height') }}:</td>
                <td>
                  {{ clientState?.frozenHeight?.revisionHeight.toString() }}
                </td>
              </tr>
              <tr>
                <td class="w-52">{{ $t('ibc.latest_height') }}:</td>
                <td>
                  {{ clientState?.latestHeight?.revisionHeight.toString() }}
                </td>
              </tr>
            </tbody>
          </table>
          <div class="md:hidden w-full border-b border-base-300 mb-2"></div>
          <table class="table table-sm text-sm w-full capitalize">
            <thead class="text-white font-semibold">
              <tr>
                <td colspan="2">{{ $t('ibc.upgrade_parameters') }}</td>
              </tr>
            </thead>
            <tbody>
              <tr>
                <td colspan="2">
                  <div class="flex justify-between">
                    <span>{{ $t('ibc.allow_update_after_expiry') }}:</span>
                    <span>{{ clientState?.allowUpdateAfterExpiry }}</span>
                  </div>
                </td>
              </tr>
              <tr>
                <td colspan="2">
                  <div class="flex justify-between">
                    <span
                      >{{ $t('ibc.allow_update_after_misbehaviour') }}:
                    </span>
                    <span>{{ clientState?.allowUpdateAfterMisbehaviour }}</span>
                  </div>
                </td>
              </tr>
              <tr>
                <td class="w-52">{{ $t('ibc.upgrade_path') }}:</td>
                <td class="text-right text-break">
                  {{ clientState?.upgradePath?.join(', ') }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
    <div class="section">
      <h2 class="card-title text-white">{{ $t('ibc.channels') }}</h2>
      <div class="overflow-auto">
        <table class="table w-full mt-4">
          <thead>
            <tr class="text-white">
              <th>{{ $t('ibc.txs') }}</th>
              <th style="position: relative; z-index: 2">
                {{ $t('ibc.channel_id') }}
              </th>
              <th>{{ $t('ibc.port_id') }}</th>
              <th>{{ $t('ibc.state') }}</th>
              <th>{{ $t('ibc.counterparty') }}</th>
              <th>{{ $t('ibc.hops') }}</th>
              <th>{{ $t('ibc.version') }}</th>
              <th class="text-right">{{ $t('ibc.ordering') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="v in ibcStore.registryChannels">
              <td>
                <div class="flex gap-1">
                  <button
                    class="btn btn-xs !bg-[rgba(39,120,77,0.20)] border border-[rgba(39,120,77,0.20)] rounded-lg !text-[#39DD47]"
                    @click="
                      fetchRecevingTxs(
                        v[ibcStore.sourceField].channel_id,
                        v[ibcStore.sourceField].port_id
                      )
                    "
                    :disabled="loading"
                  >
                    <span
                      v-if="loading"
                      class="loading loading-spinner loading-sm !text-[#39DD47]"
                    ></span>
                    {{ $t('ibc.btn_in') }}
                  </button>
                  <button
                    class="btn btn-xs !bg-[rgba(255,82,82,0.20)] border border-[rgba(255,82,82,0.20)] rounded-lg !text-[#FF5252]"
                    @click="
                      fetchSendingTxs(
                        v[ibcStore.sourceField].channel_id,
                        v[ibcStore.sourceField].port_id
                      )
                    "
                    :disabled="loading"
                  >
                    <span
                      v-if="loading"
                      class="loading loading-spinner loading-sm !text-[#FF5252]"
                    ></span>
                    {{ $t('ibc.btn_out') }}
                  </button>
                </div>
              </td>
              <td class="text-white">
                <a href="#">{{ v[ibcStore.sourceField].channel_id }}</a>
              </td>
              <td>{{ v[ibcStore.sourceField].port_id }}</td>
            </tr>
            <tr v-for="v in channels">
              <td>
                <div class="flex gap-1">
                  <button
                    class="btn btn-xs !bg-[rgba(39,120,77,0.20)] border border-[rgba(39,120,77,0.20)] rounded-lg !text-[#39DD47]"
                    @click="fetchRecevingTxs(v.channelId, v.portId)"
                    :disabled="loading"
                  >
                    <span
                      v-if="loading"
                      class="loading loading-spinner loading-sm !text-[#39DD47]"
                    ></span>
                    {{ $t('ibc.btn_in') }}
                  </button>
                  <button
                    class="btn btn-xs !bg-[rgba(255,82,82,0.20)] border border-[rgba(255,82,82,0.20)] rounded-lg !text-[#FF5252]"
                    @click="fetchSendingTxs(v.channelId, v.portId)"
                    :disabled="loading"
                  >
                    <span
                      v-if="loading"
                      class="loading loading-spinner loading-sm !text-[#FF5252]"
                    ></span>
                    {{ $t('ibc.btn_out') }}
                  </button>
                </div>
              </td>
              <td class="text-white">
                <a href="#" @click="loadChannel(v.channelId, v.portId)">{{
                  v.channelId
                }}</a>
              </td>
              <td class="text-white">{{ v.portId }}</td>
              <td>
                <div
                  class="text-xs truncate relative py-2 px-4 rounded-full w-fit text-white"
                  :class="`text-${color(v.state.toString())}`"
                >
                  <!-- <span
                    class="inset-x-0 inset-y-0 opacity-10 absolute"
                    :class="`bg-${color(v.state.toString())}`"
                  ></span> -->
                  {{ v.state }}
                </div>
              </td>
              <td>
                {{ v.counterparty?.portId }}/{{ v.counterparty?.channelId }}
              </td>
              <td class="text-white">{{ v.connectionHops.join(', ') }}</td>
              <td>{{ v.version }}</td>
              <td class="text-right text-white">{{ v.ordering }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <div v-if="channel_id">
      <h3 class="card-title capitalize">
        Transactions ({{ channel_id }} {{ port_id }} {{ direction }})
      </h3>
      <table class="table">
        <thead>
          <tr>
            <td class="text-white">{{ $t('ibc.height') }}</td>
            <td class="text-white">{{ $t('ibc.txhash') }}</td>
            <td class="text-white">{{ $t('ibc.messages') }}</td>
            <td class="text-white">{{ $t('ibc.time') }}</td>
          </tr>
        </thead>
        <tbody>
          <tr v-for="resp in txs?.txs">
            <td>{{ resp.height }}</td>
            <td>
              <div class="text-xs truncate text-primary dark:text-link">
                <RouterLink
                  :to="`/${chainStore.chainName}/tx/${toHex(resp.hash)}`"
                  >{{ toHex(resp.hash) }}</RouterLink
                >
              </div>
            </td>
            <td>
              <div class="flex">
                {{ format.messages(resp.txRaw.body.messages) }}
                <Icon
                  v-if="resp.result.code === 0"
                  icon="mdi-check"
                  class="text-success text-lg"
                />
                <Icon v-else icon="mdi-multiply" class="text-error text-lg" />
              </div>
            </td>
            <td>
              {{ resp.timestamp ? format.toLocaleDate(resp.timestamp) : '-' }}
            </td>
          </tr>
        </tbody>
      </table>
      <PaginationBar
        :limit="page.limit"
        :total="txs?.totalCount?.toString()"
        :callback="pageload"
      />
    </div>
  </div>
</template>
