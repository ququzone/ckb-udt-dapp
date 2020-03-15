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
          <el-button type="primary" @click="showTransfer(scope.row)" v-show="scope.row.udt !== '0'" size="small">Transfer UDT</el-button>
          <el-button type="primary" @click="showBurn(scope.row)" v-show="scope.row.udt !== '0'" size="small">Burn UDT</el-button>
          <el-button type="primary" @click="showIssue(scope.row)" v-show="scope.row.lock === '0x6a242b57227484e904b4e08ba96f19a623c367dcbd18675ec6f2a71a0ff4ec26'" size="small">Issue UDT</el-button>
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
        <el-button type="primary" @click="issue()">Issue</el-button>
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
        <el-button type="primary" @click="transfer()">Transfer</el-button>
      </div>
    </el-dialog>
    <el-dialog title="Burn UDT" :visible.sync="burnFormVisible">
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="burn()">Burn</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import BN from "bn.js";
import axios from "axios";
const { scriptToAddress, addressToScript } = require("@keyper/specs/lib/address");

export default {
  name: 'Locks',
  data() {
    return {
      websock: null,
      tableData: [],
      issueFormVisible: false,
      issueForm: {
        amount: 0,
        lock: '',
        meta: null,
      },
      transferFormVisible: false,
      transferForm: {
        recipient: '',
        amount: 0,
        lock: '',
        meta: null,
      },
      burnFormVisible: false,
      burnForm: {
        lock: '',
        meta: null,
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
      if (type === "event") {
        if (request.event === "SEND_TX") {
          this.issueFormVisible = false;
          this.burnFormVisible = false;
          this.transferFormVisible = false;
        }
      }
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
              meta: lock.meta,
            });
          });
        }
      }
    },
    websocketsend(data) {
      this.websock.send(data);
    },
    showIssue(row) {
      this.issueForm.lock = row.lock;
      this.issueForm.meta = row.meta;
      this.issueFormVisible = true;
    },
    showBurn(row) {
      this.burnForm.lock = row.lock;
      this.burnForm.meta = row.meta;
      this.burnFormVisible = true;
    },
    showTransfer(row) {
      this.transferForm.lock = row.lock;
      this.transferForm.meta = row.meta;
      this.transferFormVisible = true;
    },
    async issue() {
      if (this.issueForm.amount <= 0) {
        this.issueFormVisible = false;
        return;
      }
      const _cellCapacity = new BN(14200000000);
      const _fee = new BN(1000);
      const rawTx = {
        version: "0x0",
        cellDeps: [{
          outPoint: {
            txHash: "0x78fbb1d420d242295f8668cb5cf38869adac3500f6d4ce18583ed42ff348fa64",
            index: "0x0"
          },
          depType: "code",
        }],
        headerDeps: [],
        inputs: [],
        outputs: [],
        witnesses: [],
        outputsData: []
      };
      rawTx.outputs.push({
        capacity: `0x${_cellCapacity.toString(16)}`,
        lock: this.issueForm.meta.script,
        type: {
          hashType: "data",
          codeHash: "0x48dbf59b4c7ee1547238021b4869bceedf4eea6b43772e5d66ef8865b6ae7212",
          args: this.issueForm.lock,
        },
      });
      rawTx.outputsData.push(`0x${new BN(this.issueForm.amount).toArrayLike(Buffer, "le", 16).toString("hex")}`);

      const total = _cellCapacity.add(_fee);
      const result = await axios.post("http://localhost:50002", {
        lockHash: this.issueForm.lock,
        typeHash: "null",
        data: "0x",
        capacity: total.toString(),
      });
      for (let i = 0; i < result.data.cells.length; i++) {
        const element = result.data.cells[i];
      
        rawTx.inputs.push({
          previousOutput: {
            txHash: element.txHash,
            index: element.index,
          },
          since: "0x0",
        });
        rawTx.witnesses.push("0x");
      }

      const resultTotal = new BN(result.data.total, 16);
      if (resultTotal.gt(total) && resultTotal.sub(total).gt(new BN(6100000000))) {
        rawTx.outputs.push({
          capacity: `0x${resultTotal.sub(total).toString(16)}`,
          lock: this.issueForm.meta.script
        });
        rawTx.outputsData.push("0x");
      }
      rawTx.witnesses[0] = {
        lock: "",
        inputType: "",
        outputType: "",
      };
    
      const signObj = {
        target: this.issueForm.lock,
        tx: rawTx,
      };
      
      this.websocketsend(`42/keyper,["api", {"data": {"origin": "localhost", "payload":${JSON.stringify(signObj)}}, "type":"sign"}]`);
    },
    async burn() {
      const rawTx = {
        version: "0x0",
        cellDeps: [{
          outPoint: {
            txHash: "0x78fbb1d420d242295f8668cb5cf38869adac3500f6d4ce18583ed42ff348fa64",
            index: "0x0"
          },
          depType: "code",
        }],
        headerDeps: [],
        inputs: [],
        outputs: [],
        witnesses: [],
        outputsData: []
      };
      const result = await axios.post("http://localhost:50002", {
        type: "udt",
        lockHash: this.burnForm.lock,
        typeCodeHash: "0x48dbf59b4c7ee1547238021b4869bceedf4eea6b43772e5d66ef8865b6ae7212",
      });
      for (let i = 0; i < result.data.cells.length; i++) {
        const element = result.data.cells[i];

        rawTx.inputs.push({
          previousOutput: {
            txHash: element.txHash,
            index: element.index,
          },
          since: "0x0",
        });
        rawTx.witnesses.push("0x");
      }
      rawTx.outputs.push({
        capacity: `0x${new BN(result.data.totalCKB, 16).sub(new BN(1000)).toString(16)}`,
        lock: this.burnForm.meta.script,
      });
      rawTx.outputsData.push("0x");
      rawTx.witnesses[0] = {
        lock: "",
        inputType: "",
        outputType: "",
      };

      const signObj = {
        target: this.burnForm.lock,
        tx: rawTx,
      };
      
      this.websocketsend(`42/keyper,["api", {"data": {"origin": "localhost", "payload":${JSON.stringify(signObj)}}, "type":"sign"}]`);
    },
    async transfer() {
      if (this.transferForm.amount <= 0) {
        this.transferFormVisible = false;
        return;
      }
      const recipientScript = addressToScript(this.transferForm.recipient);
      const _cellCapacity = new BN(14200000000);
      const _fee = new BN(1000);
      const amount = new BN(this.transferForm.amount);
      const rawTx = {
        version: "0x0",
        cellDeps: [{
          outPoint: {
            txHash: "0x78fbb1d420d242295f8668cb5cf38869adac3500f6d4ce18583ed42ff348fa64",
            index: "0x0"
          },
          depType: "code",
        }],
        headerDeps: [],
        inputs: [],
        outputs: [],
        witnesses: [],
        outputsData: []
      };
      rawTx.outputs.push({
        capacity: `0x${_cellCapacity.toString(16)}`,
        lock: recipientScript,
        type: {
          hashType: "data",
          codeHash: "0x48dbf59b4c7ee1547238021b4869bceedf4eea6b43772e5d66ef8865b6ae7212",
          args: "0x6a242b57227484e904b4e08ba96f19a623c367dcbd18675ec6f2a71a0ff4ec26",
        },
      });
      rawTx.outputsData.push(`0x${amount.toArrayLike(Buffer, "le", 16).toString("hex")}`);
      let result = await axios.post("http://localhost:50002", {
        type: "udt",
        lockHash: this.transferForm.lock,
        typeCodeHash: "0x48dbf59b4c7ee1547238021b4869bceedf4eea6b43772e5d66ef8865b6ae7212",
        capacity: amount.toString(),
      });
      for (let i = 0; i < result.data.cells.length; i++) {
        const element = result.data.cells[i];

        rawTx.inputs.push({
          previousOutput: {
            txHash: element.txHash,
            index: element.index,
          },
          since: "0x0",
        });
        rawTx.witnesses.push("0x");
      }

      let totalCKB = _cellCapacity.add(_fee);
      let total = new BN(result.data.total, 16);
      let gatheredCKB = new BN(result.data.totalCKB, 16);
      if (total.gt(amount)) {
        rawTx.outputs.push({
          capacity: `0x${_cellCapacity.toString(16)}`,
          lock: this.transferForm.meta.script,
          type: {
            hashType: "data",
            codeHash: "0x48dbf59b4c7ee1547238021b4869bceedf4eea6b43772e5d66ef8865b6ae7212",
            args: "0x6a242b57227484e904b4e08ba96f19a623c367dcbd18675ec6f2a71a0ff4ec26",
          }
        });
        rawTx.outputsData.push(`0x${total.sub(amount).toArrayLike(Buffer, "le", 16).toString("hex")}`);
        totalCKB = totalCKB.add(_cellCapacity);
      }

      if (gatheredCKB.lt(totalCKB)) {
        result = await axios.post("http://localhost:50002", {
          lockHash: this.transferForm.lock,
          typeHash: "null",
          data: "0x",
          capacity: totalCKB.sub(gatheredCKB).toString(),
        });
        total = new BN(result.data.total, 16);
        for (let i = 0; i < result.data.cells.length; i++) {
          const element = result.data.cells[i];

          rawTx.inputs.push({
            previousOutput: {
              txHash: element.txHash,
              index: element.index,
            },
            since: "0x0",
          });
          rawTx.witnesses.push("0x");
        }

        gatheredCKB.iadd(total);
      }

      if (gatheredCKB.gt(totalCKB) && gatheredCKB.sub(totalCKB).gt(new BN(6100000000))) {
        rawTx.outputs.push({
          capacity: `0x${gatheredCKB.sub(totalCKB).toString(16)}`,
          lock: this.transferForm.meta.script
        });
        rawTx.outputsData.push("0x");
      }

      rawTx.witnesses[0] = {
        lock: "",
        inputType: "",
        outputType: "",
      };

      const signObj = {
        target: this.transferForm.lock,
        tx: rawTx,
      };
      
      this.websocketsend(`42/keyper,["api", {"data": {"origin": "localhost", "payload":${JSON.stringify(signObj)}}, "type":"sign"}]`);
    },
  }
}
</script>

