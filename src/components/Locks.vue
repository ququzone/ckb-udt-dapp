<template>
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
        <el-button @click="handleClick(scope.row)" type="text" size="small">查看</el-button>
      </template>
    </el-table-column>
  </el-table>
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
      tableData: []
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
            console.log(udt);
            this.tableData.push({
              address: scriptToAddress(lock.meta.script, {networkPrefix: "ckt", short: true}),
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

