<script lang="ts" setup>
import { useWasmStore } from '../WasmStore';
import { ref } from 'vue';
import DynamicComponent from '@/components/dynamic/DynamicComponent.vue';
import PaginationBar from '@/components/PaginationBar.vue';
import { PageRequest } from '@/types';
import { useTxDialog } from '@/stores';
import type { QueryContractsByCodeResponse } from 'cosmjs-types/cosmwasm/wasm/v1/query';
import type { ContractInfo } from 'cosmjs-types/cosmwasm/wasm/v1/types';

const props = defineProps(['code_id', 'chain']);

const pageRequest = ref(new PageRequest());
const response = ref({} as QueryContractsByCodeResponse);

const info = ref({} as ContractInfo | undefined);
const dialog = useTxDialog();
const infoDialog = ref(false);
const wasmStore = useWasmStore();
function loadContract(pageNum: number) {
  const pr = new PageRequest();
  pr.setPage(pageNum);
  if (String(props.code_id).search(/^[\d]+$/) > -1) {
    // query with code id
    wasmStore.wasmClient.getWasmCodeContracts(props.code_id, pr).then((x) => {
      response.value = x;
    });
  } else {
    // query by creator
    wasmStore.wasmClient
      .getWasmContractsByCreator(props.code_id, pr)
      .then((x) => {
        response.value = {
          contracts: x.contractAddresses,
          pagination: x.pagination,
        };
      });
  }
}
loadContract(1);

function showInfo(address: string) {
  wasmStore.wasmClient.getWasmContractInfo(address).then((x) => {
    info.value = x;
  });
}
</script>
<template>
  <div>
    <div class="section">
      <h2 class="card-title truncate w-full text-white">
        {{ $t('cosmwasm.contract_list_code') }}: {{ props.code_id }}
      </h2>
      <div class="overflow-x-auto">
        <table class="table table-compact w-full mt-4">
          <thead>
            <tr class="text-white">
              <th style="position: relative; z-index: 2">
                {{ $t('cosmwasm.contract_list') }}
              </th>
              <th>{{ $t('account.action') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="(v, index) in response.contracts"
              :key="index"
              class="hover py-4"
            >
              <td>{{ v }}</td>
              <td>
                <label
                  @click="showInfo(v)"
                  for="modal-contract-detail"
                  class="btn btn-primary btn-xs text-xs mr-2"
                  >{{ $t('cosmwasm.btn_contract') }}</label
                >
                <RouterLink
                  :to="`transactions?contract=${v}`"
                  class="btn btn-primary btn-xs text-xs"
                >
                  {{ $t('cosmwasm.btn_details') }}
                </RouterLink>
              </td>
            </tr>
          </tbody>
        </table>
        <div class="flex justify-between">
          <PaginationBar
            :limit="pageRequest.limit"
            :total="
              response.pagination?.total
                ? response.pagination.total.toString()
                : '0'
            "
            :callback="loadContract"
          />
          <label
            for="wasm_instantiate_contract"
            class="btn btn-primary my-5"
            @click="
              dialog.open('wasm_instantiate_contract', {
                codeId: props.code_id,
              })
            "
            >{{ $t('cosmwasm.instantiate_contract') }}</label
          >
        </div>
      </div>
    </div>

    <input type="checkbox" id="modal-contract-detail" class="modal-toggle" />
    <label for="modal-contract-detail" class="modal cursor-pointer">
      <label class="modal-box !w-11/12 !max-w-5xl relative p-2" for="">
        <div>
          <div class="flex items-center justify-between px-3 pt-2">
            <div class="text-lg">{{ $t('cosmwasm.contract_detail') }}</div>
            <label
              @click="infoDialog = false"
              for="modal-contract-detail"
              class="btn btn-sm btn-circle"
              >✕</label
            >
          </div>
          <div>
            <DynamicComponent :value="info" />
          </div>
        </div>
      </label>
    </label>
  </div>
</template>
