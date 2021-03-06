<template>
  <div class="instance">
    <b-container>
      <b-row>
        <b-col>
          <b-input-group prepend="Instance" class="address-input">
            <b-form-input v-model="new_address" type="text" placeholder="Other Instance"></b-form-input>
            <b-form-select v-model="new_address" :options="network.public.instances" value-field="''" />
            <b-input-group-append>
              <b-btn variant="primary" @click="update(new_address)">Update</b-btn>
              <b-btn variant="secondary" v-b-modal.modal-create-instance>New</b-btn>
            </b-input-group-append>
          </b-input-group>
          <b-modal id="modal-create-instance" title="Create Instance" @ok="createInstance" @shown="clearCreateInstance">
            <b-form>
              <b-form-group label="Name" label-for="name">
                <b-form-input id="name" type="text" v-model="newInstance.name" placeholder=""/>
              </b-form-group>
              <b-form-group label="Account" label-for="account">
                <b-form-select v-model="newInstance.account" :options="accounts">
                  <option value="" disabled> Please Select Account</option>
                </b-form-select>
              </b-form-group>
              <b-form-group label="Sample Procedure" label-for="entry_proc_address">
                <b-form-select v-model="newInstance.test_choice">
                  <option value="" disabled> Please Select Test Entry</option>
                  <optgroup label="Procedure Object Capabilities">
                    <option :value="entry_tests.proc.call">Call Procedure</option>
                  </optgroup>
                  <optgroup label="Storage Object Capabilities">
                    <option :value="entry_tests.store.write">Write to Storage</option>
                  </optgroup>
                  <optgroup label="Log Object Capabilities">
                    <option :value="entry_tests.log.write">Write to Log</option>
                  </optgroup>
                </b-form-select>
              </b-form-group>
            </b-form>
          </b-modal>
        </b-col>
      </b-row>
      <b-row>
        <b-col>
          <b-card header="Interface">
              <b-input-group v-for="call in interface" class="abi_call">
                <b-input-group-text slot="prepend">
                  {{ call.proc_id }}
                </b-input-group-text>
                <b-input-group-text slot="prepend" v-if="call.fn.name">
                  {{ call.fn.name }}
                </b-input-group-text>
                <b-form-input id="call" v-for="input in call.fn.inputs" :key="input.name" :placeholder="input.type" v-if="call.fn.inputs.length > 0"></b-form-input>
                <b-form-input v-if="call.fn.type === 'fallback'" placeholder="Fallback" disabled></b-form-input>
                <b-input-group-append>
                  <b-btn variant="primary" @click="makeCall(call.proc_id, call.fn.name)" @mouseover="highlight = call.proc_id" @mouseout="highlight = ''">Call</b-btn>
                </b-input-group-append>
              </b-input-group>
          </b-card>
        </b-col>
        <b-col>
          <b-card no-body>
            <div slot="header" class="d-flex justify-content-between"> Procedures <b-button size="sm" class="clear-button" v-b-modal.modal-create-proc>+</b-button></div>
            <b-modal id="modal-create-proc" title="Create Procedure" @ok="createProc" @shown="clearCreateProc">
              <b-form>
                <b-form-group>
                  <b-form-input placeholder="Name" v-model="newProc.name"></b-form-input>
                  <b-form-select placeholder="Enter Address" v-model="newProc.address" :options="public_procedures">
                    <template slot="first">
                      <option :value="null" disabled>-- Please select an option --</option>
                    </template>
                  </b-form-select>
                  <b-form-input placeholder="Enter Bytecode" v-model="newProc.bytecode"></b-form-input>
                  <b-form-input placeholder="Enter Abi" v-model="newProc.abi"></b-form-input>
                  <b-form-input placeholder="Capabilities" v-model="newProc.caps"></b-form-input>
                </b-form-group>
              </b-form>
            </b-modal>
            <b-list-group flush>
              <b-list-group-item v-for="(proc, i) in procedures" :key="proc.id" class="flex-column align-items-start"  v-bind:class="{highlighted: true, highlight: highlight === proc.id, lowhighlight: highlight, callerhighlight: proc.id == 'Entry' && highlight != 'Entry' && highlight}">
                <div class="d-flex w-50 flex-column">
                  <h5 class="mb-1">{{ proc.id }}</h5>
                </div>
                <b-input-group size="sm">
                  <b-form-input readonly :value="proc.address"></b-form-input>
                  <b-input-group-append size="sm">
                    <b-btn size="sm" text="Button" variant="success"  @click="copyAddress(proc.address)">Copy</b-btn>
                  </b-input-group-append>
                </b-input-group>
                <b-input-group v-for="(abi, id) in proc.abi" class="abi_call">
                  <b-input-group-text slot="prepend" v-if="abi.name">
                    {{ abi.name }}
                  </b-input-group-text>
                  <b-form-input id="call" v-for="input in abi.inputs" :key="input.name" :placeholder="input.type"></b-form-input>
                  <b-form-input v-if="abi.type === 'fallback'" placeholder="Fallback" disabled></b-form-input>
                  <b-form-input v-else-if="abi.inputs.length == 0" placeholder="No Parameters" disabled></b-form-input>
                </b-input-group>
                <b-button size="sm" @click="removeProc(proc.id)">Remove</b-button>
              </b-list-group-item>
            </b-list-group>
          </b-card>
        </b-col>
        <b-col class="resources" tag="ul">
          <b-card no-body tag="li">
            <div slot="header" class="d-flex justify-content-between"> Storage <b-button size="sm" class="clear-button" v-b-modal.modal-create-storecap>+</b-button></div>
            <b-modal id="modal-create-storecap" title="Add Store" @ok="createStoreCap" @shown="clearCreateStoreCap">
              <b-form>
                <b-form-group>
                  <label for="cap-owner">For Procedure</label>
                  <b-form-select id="cap-owner" placeholder="Owner" v-model="newStoreCap.for" :options="procedures.map( p => ({ value: p.id, text: p.id }))"></b-form-select>
                  <label for="cap-store-start">Location</label>
                  <b-form-input id="cap-store-start" type="number" v-model="newStoreCap.start"></b-form-input>
                  <label for="cap-store-size">Size</label>
                  <b-form-input id="cap-store-size" type="number" v-model="newStoreCap.size"></b-form-input>
                </b-form-group>
              </b-form>
            </b-modal>
            <b-list-group flush>
              <b-list-group-item v-for="(store, i) in storage_caps" class="d-flex flex-column align-items-start" :key="i" v-bind:class="{highlighted: true, lowhighlight: store.owners.includes(highlight), basehighlight: highlight != '' && highlight != 'Entry'}">
                <div class="d-flex w-50 flex-column">
                  <small v-for="owner in store.owners">{{ owner }}</small>
                </div>
                <b-card no-body class="w-100">
                  <b-list-group flush>
                    <b-list-group-item v-for="(data, i) in store.data" class="d-flex justify-content-between">
                      <small>
                        {{ store.address + i}}
                      </small>
                      <span>
                        {{ data }}
                      </span>
                    </b-list-group-item>
                  </b-list-group>
                  <b-button size="sm" @click="removeCap(store)">Remove</b-button>
                </b-card>
              </b-list-group-item>
            </b-list-group>
          </b-card>
        </b-col>
        <b-col>
          <b-card no-body tag="li">
             <div slot="header" class="d-flex justify-content-between"> Logs <b-button size="sm" class="clear-button" v-b-modal.modal-create-logcap>+</b-button></div>
            <b-modal id="modal-create-logcap" title="Add Store" @ok="createLogCap" @shown="clearCreateLogCap">
              <b-form>
                <b-form-group>
                  <label for="cap-owner">For Procedure</label>
                  <b-form-select id="cap-owner" placeholder="Owner" v-model="newLogCap.for" :options="procedures.map( p => ({ value: p.id, text: p.id }))"></b-form-select>
                  <label for="cap-log-topics">Topic</label>
                  <b-input-group id="cap-log-topics">
                    <b-form-input type="text"  v-for="(topic, i) in newLogCap.topic" :key="i" placeholder="Topic " v-model="newLogCap.topic[i]"></b-form-input>
                  </b-input-group>
                </b-form-group>
              </b-form>
            </b-modal>
            <b-list-group flush>
              <b-list-group-item v-for="(log, i) in log_caps" :key="i" class="d-flex flex-column align-items-start" v-bind:class="{highlighted: true, lowhighlight: log.owners.includes(highlight), basehighlight: highlight != '' && highlight != 'Entry'}">
                <div class="d-flex w-50 flex-row">
                  <small v-for="owner in log.owners" :key='owner'>{{ owner }}</small>
                </div>
                <b-card no-body class="w-100" v-if="log.topics.length > 0">
                  <b-list-group flush>
                    <b-list-group-item v-for="(topic, i) in log.topics.slice(0)" :key="i" class="d-flex justify-content-between">
                      <small>
                        Topic {{ i + 1 }}
                      </small>
                      <span >
                        {{ topic == "\u0000" ? "*": topic }}
                      </span>
                      <b-badge> {{ logs.filter(l => l.raw.topics.length === log.topics.length && l.raw.topics.some((val, i) => log.topics[i] === val)).length }}</b-badge>
                    </b-list-group-item>
                  </b-list-group>
                </b-card>
                <span v-else>
                  Any Topics
                  <b-badge> {{ logs.length }}</b-badge>
                </span>
                <b-button size="sm" @click="removeCap(log)">Remove</b-button>
              </b-list-group-item>
            </b-list-group>
          </b-card>
        </b-col>
      </b-row>
    </b-container>
  </div>
</template>

<script <script lang="ts">
import Vue from "vue";
import Component from "vue-class-component";
import * as Store from "vuex-class";

import Navbar from "@/components/Navbar.vue";
import {
  web3,
  TestAbi,
  MIN_GAS,
  MIN_GAS_PRICE,
  ProcedureTable,
  WriteCap,
  LogCap,
  CallCap,
  Capability,
  CapabilityType,
LocalKernelAbi
} from "@/web3";
import { Network, actions } from "@/store/modules/network";
import { ActionMethod, Action } from "vuex";
import Contract from "web3/eth/contract";
import ABI, { ABIDefinition } from "web3/eth/abi";
import { EventLog } from 'web3/types';

@Component<Instance>({
  watch : {
    new_address(new_addr, old_addr) {
      if (new_addr !== old_addr) this.update(new_addr)
    }
  }
})
export default class Instance extends Vue {
  @Store.State network: Network;

  @Store.Action("network/deploy_instance") deployInstance: (account?: string) => Promise<void>;
  @Store.Action("network/update_instance") updateInstance: (instance?: {account?: string, address?: string}) => Promise<void>;
  @Store.Action("network/send_call") sendCall: (call: {proc_name: string, abi: ABIDefinition, instance: Contract}) => Promise<void>
  @Store.Action("network/deploy_procedure") deployProcedure: (proc: {name: string, abi: any}) => Promise<void>;
  @Store.Action("network/register_procedure") registerProcedure: (proc: {name: string, address: string, caps: Capability[] }) => Promise<void>
  @Store.Action("network/remove_procedure") removeProcedure: (name: string) => Promise<void>
  
  instance_address: string;
  new_address: string = '';

  highlight: string = '';
  entry_tests = TestAbi;

  newInstance = {
    name: "",
    account: "",
    proc_address: "",
    test_choice: {}
  };

  newCall = {
    abi_id: 0,
    input: []
  };

  newProc = {
    name: '',
    address: '',
    bytecode: '',
    abi: '',
    caps: [],
  }

  newStoreCap = {
    for: '',
    start: 0,
    size: 0,
  }

  newLogCap = {
    for: '',
    topic: ['', '', '', '']
  }

  entry_fn: { text: string; value: number }[] = [];
  caps: (WriteCap | CallCap | LogCap)[] = [];
  logs: EventLog[] = [];
  
  async mounted() {
    // Create New Instance First
    await this.$store.dispatch('network/connect')
    await this.deployInstance(this.network.accounts[0].id);
    this.new_address = this.instance_address || '';
    this.$store.subscribe(mutation => {
      if (mutation.type == 'network/instance_log') {
        this.logs.push(mutation.payload)
      }
    })

    await this.update(this.instance_address)
  }

  async update(address?: string) {
    if (address && !web3.utils.isAddress(address)) return;
    await this.updateInstance({address})
    await this.updateCapTable();
  }

  async copyAddress(addr: string) {
    await (navigator as any).clipboard.writeText(addr)
  }

  async createInstance() {
    let account = this.newInstance.account;
    await this.deployInstance(account);

    let entry_code = this.newInstance.test_choice;
    let name = this.newInstance.name

    await this.deployProcedure({
      name,
      abi: this.newInstance.test_choice
    })
    
    let procedures = this.network.procedures;
    let procedure = procedures[procedures.length - 1];

    const cap1 = new WriteCap(0x8500, 2);
    const cap2 = new WriteCap(0x8000, 2);

    this.registerProcedure({
      name,
      address: procedure.contract.options.address,
      caps: [cap1, cap2]
    })

    this.new_address = this.network.instance.contract.options.address;
    this.update();
  }

  clearCreateInstance() {
    this.newInstance = {
      name: "",
      account: "",
      proc_address: "",
      test_choice: {}
    };
  }

  async removeCap(cap: Capability) {
    await this.updateInstance();

    let tasks = cap.owners.map(async proc_id => {
      let key = (web3.utils.toHex(proc_id) as any ).padEnd(50, '0');
      let proc = this.network.instance.proc_table!.table[key];

      let id = proc.caps.findIndex(c => c.type === cap.type && cap.raw_values.join('') === c.raw_values.join(''))
      let new_caps = proc.caps.slice()
      new_caps.splice(id, 1);

      return await Promise.all([
        await this.removeProcedure(proc_id),
        await this.registerProcedure({
          name: proc_id,
          address: proc.location,
          caps: new_caps
        })
      ])
    })

    await Promise.all(tasks)
    await this.updateInstance();
    await this.updateCapTable();
  }

  async createLogCap() {
    let key = (web3.utils.toHex(this.newLogCap.for) as any ).padEnd(50, '0');
    let proc = this.network.instance.proc_table!.table[key];
    let len = this.newLogCap.topic.reduce((p, t, i) => p = t === '' ? p : i, -1) + 1
    let cap = new LogCap(this.newLogCap.topic.slice(0, len)) 
    let caps = (proc.caps as any).concat(new LogCap(this.newLogCap.topic.slice(0, len)))

    await this.removeProcedure(this.newLogCap.for)
    await this.registerProcedure({
      name: this.newLogCap.for,
      address: proc.location,
      caps,
    })

    await this.updateInstance();
    await this.updateCapTable();

  }

  clearCreateLogCap() {
    this.newLogCap = { for: '', topic: ['','','',''] }
  }
  // TODO: Hacky way to add a cap...
  async createStoreCap() {
    let key = (web3.utils.toHex(this.newStoreCap.for) as any ).padEnd(50, '0');
    let proc = this.network.instance.proc_table!.table[key];
    let caps = (proc.caps as any).concat(new WriteCap(this.newStoreCap.start, this.newStoreCap.size))

    await this.removeProcedure(this.newStoreCap.for)
    await this.registerProcedure({
      name: this.newStoreCap.for,
      address: proc.location,
      caps,
    })

    await this.updateInstance();
    await this.updateCapTable();
  }

  clearCreateStoreCap() {
    this.newStoreCap = { for: '', start: 0, size: 0 }
  }

  async removeProc(id: string) {
    await this.removeProcedure(id);
    await this.update();
  }
  
  async createProc() {

    let name = this.newProc.name;
    if (this.newProc.address) {

      await this.registerProcedure({
        name,
        address: this.newProc.address,
        caps: this.newProc.caps,
      })
      
    } else if (this.newProc.abi) {

      await this.deployProcedure({
        name,
        abi: JSON.parse(this.newProc.abi)
      })

      let procedures = this.network.procedures;
      let procedure = procedures[procedures.length - 1];

      await this.registerProcedure({
        name,
        address: procedure.contract.options.address,
        caps: this.newProc.caps
      })
    }

    await this.update();
  }

  clearCreateProc() {
    this.newProc = { name: '', address: '', bytecode: '', abi: '', caps: []};
  }

  async makeCall(proc_id: string, fnName: string) {
    let proc = this.procedures.find(({id}) => id === proc_id)
    let abi = proc!.abi.find(fn => fn.name === fnName)!
    
    await this.sendCall({
      proc_name: proc!.id,
      abi,
      instance: this.instance,
    })

    let events = this.network.instance.contract.events.allEvents()

    await this.updateInstance()
    await this.updateCapTable();
  }

  async getProcedureAddress(id: string, account: string) {
    let name = web3.utils.isHex(id) ? id : web3.utils.toHex(id);
    return this.instance.methods.getProcedure(name).call();
  }

  async updateCapTable() {
    let proc_table = this.network.instance.proc_table!.table!

    let result: Capability[] = [];

    for (let key in proc_table) {
      let proc = proc_table[key];

      proc.caps.forEach(async cap => {
        // Set Key as Ascii
        let name = web3.utils.hexToUtf8(key);
        // Add Owner to Cap
        cap.owners.push(name);

        // If Storage Cap, populate data with storage values
        if(cap.type === CapabilityType.StorageWrite) {
          let { address, size} = cap as WriteCap;
          let data = Array.from({length: size}, (x,i) => web3.eth.getStorageAt(this.instance.options.address, address + i).then(web3.utils.hexToNumberString));
          cap.data = await Promise.all(data) as any
        }
        
        let dup_i = result.findIndex(p_cap =>
            cap.type === p_cap.type && 
            cap.raw_values.length === p_cap.raw_values.length &&
            cap.keyValues().every((val, i) => p_cap.keyValues()[i] === val)
        )

        // If a Duplicate Cap is found, add Proc to Owners, if not add to results.
        if (dup_i !== -1 && result[dup_i].owners.indexOf(name) === -1) {
          result[dup_i].owners.push(name);
        } else {
          result.push(cap);
        }
      });
    }

    this.caps = result as any;
  }

  getProcCallers(id: string) {
    let callers: Set<string> = new Set();
    this.caps.forEach(cap => {
      if (cap instanceof CallCap) {
        if (cap.keys.length == 0 || (cap.keys as any).includes(id)) {
          cap.owners.forEach(owner => callers.add(owner))
        }
      }
    })
    return callers;
  }

  getAbi(address: string): ABIDefinition[] {
    let proc = this.network.public.procedures[address]
    if (proc && proc.abi) return proc.abi;
    return [];
  }

  getCapTypeName(num: CapabilityType) {
    switch (num) {
      case CapabilityType.ProcedureCall:
        return "Procedure Call";
      case CapabilityType.StorageWrite:
        return "Storage Write";
      case CapabilityType.LogWrite:
        return "Log Write";
      default:
        "Unknown";
    }
  }

  getAbiFunctions(abi: ABIDefinition[]) {
    return abi.map((fn, id) => {
      let name = fn.type === "fallback" ? "fallback" : fn.name;
      let inputs = !fn.inputs
        ? ""
        : fn.inputs.map(input => input.type + " " + input.name).join(",");
      let outputs =
        !fn.outputs || fn.outputs.length === 0
          ? ""
          : "-> " + fn.outputs.map(output => output.type).join(",");
      return {
        value: id,
        text: `${name}(${inputs})${outputs}`
      };
    });
  }

  get procedures():  { id: string; address: string, abi: ABIDefinition[], owners: string[] }[] {

    const proc_table: ProcedureTable = this.network.instance.proc_table!;
    let table_keys = Object.keys(proc_table.table);

    return table_keys.map(hex_id => {
      let proc = proc_table.table[hex_id]
      let id = web3.utils.hexToUtf8(hex_id);
      let address = proc.location
      let abi: ABIDefinition[] = this.getAbi(address);
      let owners = Array.from(this.getProcCallers(id))
      return { id, address, abi, owners };
    })
  }

  get public_procedures(): string[] {
    return Object.keys(this.network.public.procedures)
  }

  get interface() {
    return this.procedures.reduce((res, proc, proc_i) => {
      return res.concat(proc.abi.map((fn, id) => ({ id, proc_i, proc_id: proc.id, fn })).filter(x => x.fn.type !== 'fallback'))
    }, [] as {fn: ABIDefinition, proc_i: number, id: number, proc_id: string}[])
  }
  get storage_caps(): WriteCap[] {
    return this.caps.filter(cap => cap.type === CapabilityType.StorageWrite) as WriteCap[]
  }

  get log_caps(): LogCap[] {
    return this.caps.filter(cap => cap.type === CapabilityType.LogWrite) as LogCap[]
  }

  get instance(): Contract {
    return this.network.instance.contract
  }

  get accounts() {
    let accounts: Network["accounts"] = this.network.accounts;
    return accounts.map(({ id }) => id);
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.instance {
  padding-top: 2rem;
}

.address-input {
  margin-bottom: 1rem;
}
.abi_call {
  margin-top: 0.5rem;
}

.highlighted {
  background-color: #fff;
}

.highlight, .lowhighlight, .callerhighlight, .highlighted {
  transition: background-color 0.5s cubic-bezier(0.23, 1, 0.320, 1);
}

.highlight {
  color: #fff;
  background-color:rgb(107, 107, 107);
}
.highlight.lowhighlight {
    color: #fff;
    background-color:rgb(73, 73, 73);
}

.callerhighlight {
  background-color: #ddd;
}

.lowhighlight {
  background-color: #aaa;
}
.basehighlight {
  background-color: #ddd;
}

.lowhighlight.basehighlight {
    background-color: #aaa;
}

.resources li {
  margin-bottom: 2rem;
}

.clear-button {
  padding: 0 5px;
  background-color: transparent;
  color: #333;
}
</style>