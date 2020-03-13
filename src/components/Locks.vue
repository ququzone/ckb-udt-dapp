<template>
  <div>
    <el-table
      :data="tableData"
      border
      style="width: 100%">
      <el-table-column
        prop="address"
        label="Address"
        width="500">
      </el-table-column>
      <el-table-column
        prop="ckb"
        label="CKB"
        width="300">
      </el-table-column>
      <el-table-column
        prop="udt"
        label="UDT"
        width="300">
      </el-table-column>
      <el-table-column
        label="Operations">
        <template slot-scope="scope">
          <el-button type="primary" @click="issueFormVisible = true" :disabled="scope.row.lock !== '0x6a242b57227484e904b4e08ba96f19a623c367dcbd18675ec6f2a71a0ff4ec26'" size="small">Issue UDT</el-button>
          <el-button type="primary" @click="transferFormVisible = true" :disabled="scope.row.udt === '0'" size="small">Transfer UDT</el-button>
          <el-button type="primary" @click="burnFormVisible = true" :disabled="scope.row.udt === '0'" size="small">Burn UDT</el-button>
        </template>
      </el-table-column>
    </el-table>
    <el-dialog title="Issue UDT" :visible.sync="issueFormVisible">
      <el-form :model="issueForm">
        <el-form-item label="Amount" label-width="120px">
          <el-input v-model="issueForm.amount" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="issueFormVisible = false">Issue</el-button>
      </div>
    </el-dialog>
    <el-dialog title="Transfer UDT" :visible.sync="transferFormVisible">
      <el-form :model="transferForm">
        <el-form-item label="Recipient" label-width="120px">
          <el-input v-model="transferForm.recipient" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="Amount" label-width="120px">
          <el-input v-model="transferForm.amount" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="transferFormVisible = false">Transfer</el-button>
      </div>
    </el-dialog>
    <el-dialog title="Burn UDT" :visible.sync="burnFormVisible">
      <el-form :model="burnForm">
        <el-form-item label="Amount" label-width="120px">
          <el-input v-model="burnForm.amount" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="burnFormVisible = false">Burn</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import BN from "bn.js";
import axios from "axios";
const { scriptToAddress } = require("@keyper/specs/lib/address");

export default {
  name: 'Locks',
  data() {
    return {
      websock: null,
      tableData: [],
      issueFormVisible: false,
      issueForm: {
        amount: 0,
      },
      transferFormVisible: false,
      transferForm: {
        recipient: '',
        amount: 0,
      },
      burnFormVisible: false,
      burnForm: {
        amount: 0,
      },
    }
  },
  created() {
    this.initWebSocket();
  },
  methods: {
    initWebSocket() {
      this.websock = new WebSocket('ws://localhost:50001');
      this.websock.onmessage = this.websocketonmessage;
      this.websock.onopen = this.websocketonopen;
    },
    websocketonopen() {
      this.websocketsend(`42/keyper,["api", {"data": {"origin": "localhost", "payload": {"method": "ALL_LOCKS"}}, "type":"query"}]`);
    },
    async websocketonmessage(e) {
      if (e.data.indexOf('42/keyper') === -1) {
        return false;
      }
      const [type, request] = JSON.parse(e.data.replace('42/keyper,', ''));
      if (type === "api") {
        if (request.query === "ALL_LOCKS") {
          const locks = request.payload;

          locks.forEach(async lock => {
            const ckb = await axios.post("http://localhost:50002", {
              lockHash: lock.hash,
            });
            const udt = await axios.post("http://localhost:50002", {
              type: "udt",
              typeHash: "0x2c0da3548618bc98003075f2deabd3569c4c4a1a55e63b2e7677aeed9c45c2b7",
              lockHash: lock.hash,
            });
            this.tableData.push({
              address: scriptToAddress(lock.meta.script, {networkPrefix: "ckt", short: true}),
              lock: lock.hash,
              ckb: new BN(ckb.data.total, 16).toString(),
              udt: new BN(udt.data.total, 16).toString(),
            });
          });
        }
      }
    },
    websocketsend(data) {
      this.websock.send(data);
    },
  }
}
</script>

