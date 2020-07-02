<template>
  <div>
    <el-tabs v-if="interfaces.length > 0" v-model="editorIface" type="border-card" @tab-click="update_info" closable @edit="pre_del">
      <el-tab-pane v-for="iface in interfaces" :key="iface.name" :name="iface.name" :label="iface.name">
        <el-table :data="info">
          <el-table-column :label="$t('Status')">
            <template v-slot="{ row }">
              <strong>{{ $t('Uptime') }}</strong>: {{ row.isUp() ? '%t'.format(row.getUptime()) : $t('Interface is down') }}<br/>
              <strong>RX</strong>: {{ '%mB'.format(row.getStatistics().rx_bytes) }}<br/>
              <strong>TX</strong>: {{ '%mB'.format(row.getStatistics().tx_bytes) }}<br/>
            </template>
          </el-table-column>
          <el-table-column :label="$t('Information')">
            <template v-slot="{ row }">
              <strong>IPv4</strong>: {{ row.getIPv4Addrs().join(',') }}<br/>
              <strong>IPv6</strong>: {{ row.getIPv6Addrs().join(',') }}<br/>
              <strong>MAC</strong>: {{ row.getDevice() ? row.getDevice().macaddr : '' }}<br/>
            </template>
          </el-table-column>
          <el-table-column :label="$t('Edit')">
            <template v-slot="{ row }">
              <div class="edit">
                <el-button size="mini" @click="ifup(row.name)">{{ $t('Start') }}</el-button>
                <el-button size="mini" @click="ifdown(row.name)">{{ $t('Stop') }}</el-button><br />
                <p></p>
              </div>
              <div class="edit">
                <el-button type="primary" size="mini" @click="edit(row.name)">{{ $t('Edit') }}</el-button>
                <el-button type="danger" size="mini" @click="del(row.name)">{{ $t('Delete') }}</el-button>
              </div>
            </template>
          </el-table-column>
        </el-table>
      </el-tab-pane>
      <el-button type="primary" size="small" style="margin-top: 10px" @click="handleAdd">+ {{ $t('Add interface') }}</el-button>
    </el-tabs>
    <el-dialog :title="dialogTitle" :visible.sync="dialogVisible" custom-class="interface-edit-dialog" :before-close="handleClose">
      <uci-form config="network" v-if="dialogVisible" :apply-timeout="15">
        <uci-section :name="editorIface">
          <uci-tab :title="$t('General Settings')" name="general">
            <uci-option-switch :label="$t('Start on boot')" name="auto" initial="1"></uci-option-switch>
            <uci-option-list :label="$t('Protocol')" name="proto" :options="protocols" initial="none" required @change="protoChanged"></uci-option-list>
          </uci-tab>
          <uci-tab :title="$t('Advanced Settings')" name="advanced">
            <uci-option-switch :label="$t('Use builtin IPv6-management')" name="delegate" initial="1"></uci-option-switch>
            <uci-option-switch :label="$t('Force link')" name="force_link" :initial="proto === 'static' ? true : false" :description="$t('Set interface properties regardless of the link carrier (If set, carrier sense events do not invoke hotplug handlers).')"></uci-option-switch>
          </uci-tab>
          <component v-if="proto !== '' && proto !== 'none'" :is="'proto-' + proto" @mounted="onProtoMounted"></component>
        </uci-section>
      </uci-form>
    </el-dialog>
  </div>
</template>
<script>

import ProtoDhcp from './proto/dhcp.vue'
import ProtoStatic from './proto/static.vue'
import ProtoPppoe from './proto/pppoe.vue'
import ProtoPptp from './proto/pptp.vue'
import ProtoL2tp from './proto/l2tp.vue'
import Proto3g from './proto/3g.vue'
import Ifname from './ifname.vue'

export default {
  data() {
    return {
      virtual: false,
      floating: false,
      interfaces: [],
      devices: [],
      zones: [],
      editorIface: 'lan',
      proto:'',
      info:[],
      dialogVisible:false,
      protocols: [
        ['none', this.$t('Unmanaged')],
        ['dhcp', this.$t('DHCP Client')],
        ['static', this.$t('Static address')],
        ['pppoe', 'PPPoE'],
        ['pptp', 'PPtP'],
        ['l2tp', 'L2TP'],
        ['3g', '3G']
      ]
    }
  },
  components: {
    ProtoDhcp,
    ProtoStatic,
    ProtoPppoe,
    ProtoPptp,
    ProtoL2tp,
    Proto3g,
    Ifname
  },
  computed: {
    dialogTitle() {
      return `${this.$t('Configure')} "${this.editorIface}"`
    }
  },
  timers: {
    upgrade_info: {time: 3000, autostart: true, immediate: true, repeat: true}
  },
  methods: {
    pre_del(name, action) {
      if (action === 'remove') {
        this.del(name);
      }
    },
    handleClose(done) {
      this.upgrade_info();
      done();
    },
    onProtoMounted(proto) {
      this.virtual = proto.virtual;
      this.floating = proto.floating;
    },
    update_info(tab) {
      this.editorIface = tab.name;
      this.$network.load().then(() => {
        this.info = this.$network.getInterfaces().filter(d => d.name === tab.name);
      })
    },
    protoChanged(proto) {
      this.proto = proto;
    },
    upgrade_info() {
      this.$network.load().then(() => {
        this.interfaces = this.$network.getInterfaces();
        this.info = this.$network.getInterfaces().filter(d => d.name === this.editorIface);
      })
    },
    saveType(sid, value) {
      this.$uci.set('network', sid, 'type', value || '');
    },
    loadZone() {
      return new Promise(resolve => {
        this.$firewall.load().then(() => {
          this.zones = this.$firewall.zones.map(z => z.name());
          const z = this.$firewall.findZoneByNetwork(this.editorIface);
          if (z)
            resolve(z.name());
          resolve();
        });
      });
    },
    saveZone(sid, value) {
      let z = this.$firewall.findZoneByNetwork(this.editorIface);

      if (!value) {
        if (z)
          z.delNetwork(this.editorIface);
        return;
      }

      if (z) {
        if (value === z.name())
          return;
        z.delNetwork(this.editorIface);
      }

      z = this.$firewall.findZoneByName(value);
      if (!z)
        z = this.$firewall.createZone(value);
      z.addNetwork(this.editorIface);
    },
    edit(iface) {
      this.editorIface = iface;
      this.dialogVisible = true;
    },
    ifup(name) {
      this.$ubus.call('oui.network', 'ifup', {name: name});
    },
    ifdown(name) {
      this.$ubus.call('oui.network', 'ifdown', {name: name});
    },
    del(name) {
      this.$confirm(this.$t('Really delete this interface? The deletion cannot be undone!You might lose access to this device if you are connected via this interface.'), `${this.$t('Delete interface')} "${name}"`).then(() => {
        const loading = this.$getLoading();

        this.$uci.del('network', name);
        this.$uci.save().then(() => {
          this.$uci.apply().then(() => {
            this.editorIface = 'lan';
            this.upgrade_info();
            loading.close();
          });
        });
      });
    },
    add(name) {
      const loading = this.$getLoading();

      this.$uci.add('network', 'interface', name);
      this.$uci.save().then(() => {
        this.$uci.apply().then(() => {
          loading.close();
          this.upgrade_info();
          this.editorIface = name;
          this.dialogVisible = true;
        });
      });
    },
    handleAdd() {
      this.$prompt(this.$t('Please input a name'), this.$t('Add'), {
        inputValidator: value => {
          if (!value || value.match(/^[a-zA-Z0-9_]+$/) === null)
            return this.$t('Must be a valid UCI identifier');

          for (let i = 0; i < this.interfaces.length; i++)
            if (this.interfaces[i].name === value)
              return this.$t('Name already used');

          return true;
        }
      }).then(r => {
        this.add(r.value);
      });
    }
  },

  created() {
    this.$network.load().then(() => {
      this.interfaces = this.$network.getInterfaces();
      this.info = this.$network.getInterfaces().filter(d => d.name === 'lan');
    });
  }
}

</script>

<style lang="scss">
.interface-edit-dialog {
  .el-dialog__header {
    padding: 10px 20px 10px;
  }
  .el-dialog__body {
    padding: 0;
  }
}
</style>
