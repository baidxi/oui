<template>
 <div>
    <el-table :data="info">
        <el-table-column :label="$t('Status')">
            <template v-slot="{ row }">
                <strong>{{ $t('Uptime') }}</strong>: {{ row.isUp() ? '%t'.format(row.getUptime()) : $t('Interface is down') }}<br/>
                <strong>MAC</strong>: {{ row.getDevice() ? row.getDevice().macaddr : '' }}<br/>
                <strong>RX</strong>: {{ '%mB'.format(row.getStatistics().rx_bytes) }}<br/>
                <strong>TX</strong>: {{ '%mB'.format(row.getStatistics().tx_bytes) }}<br/>
                <strong>IPv4</strong>: {{ row.getIPv4Addrs().join(',') }}<br/>
                <strong>IPv6</strong>: {{ row.getIPv6Addrs().join(',') }}<br/>
            </template>
        </el-table-column>
    </el-table>
 </div>
</template>

<script>

export default {
  data() {
    return {
      proto:'static',
      virtual:false,
      floating:false,
      info:[]
    }
  },

  methods:{
    onProtoMounted(proto) {
      this.virtual = proto.virtual;
      this.floating = proto.floating;
    }
  },
  created() {
    this.$network.load().then(() => {
      this.info = this.$network.getInterfaces().filter(d => d.name === 'wan');0
    })
  }
}
</script>

<style>
</style>
