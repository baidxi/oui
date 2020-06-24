<template>
    <uci-form config="sqm">
        <uci-section type="queue" :title="$t('Smart Queue Management')">
            <uci-tab :title="$t('Basic Settings')" name="tab_basic">
                <uci-option-switch :label="$t('Enable this SQM instance.')" name="enabled"></uci-option-switch>
                <uci-option-list :label="$t('Interface name')" name="interface" :options="devices"></uci-option-list>
                <uci-option-input :label="$t('Download')" name="download" :rules="{type: 'uinteger', min: 0}" append="kbit/s" :description="$t('Download speed (kbit/s) (ingress) set to 0 to selectively disable ingress shaping')"></uci-option-input>
                <uci-option-input :label="$t('Upload')" name="upload" :rules="{type: 'uinteger', min:0 }" append="kbit/s" :description="$t('Upload speed (kbit/s) (egress) set to 0 to selectively disable egress shaping')"></uci-option-input>
            </uci-tab>
            <uci-tab :title="$t('Queue Discipline')" name="tab_qdisc">
                <uci-option-list :label="$t('qdisk')" :options="qdisk" @change="qdisk_change" name="qdsik" initial="Default" :description="$t('Queuing disciplines useable on this system. After installing a new qdisc, you need to restart the router to see updates')"></uci-option-list>
                <uci-option-list :label="$t('Script')" name="script" @change="script_change" :options="scripts" :description="$t('Queue setup script')"></uci-option-list>
                <uci-option-switch :label="$t('Advanced Settings')" name="qdisc_advanced" :description="$t('Show and Use Advanced Configuration. Advanced options will only be used as long as this box is checked.')"></uci-option-switch>
                <uci-option-list :label="$t('DSCP')" name="squash_dscp" :options="squash_dscp" initial="1" depend="qdisc_advanced == 1" :description="$t('Squash DSCP on inbound packets (ingress)')"></uci-option-list>
                <uci-option-list :label="$t('Ingress')" name="squash_ingress" initial="1" :options="squash_ingress" depend="qdisc_advanced == 1" :description="$t('Ignore DSCP on ingress')"></uci-option-list>
                <uci-option-list :label="$t('Ingress ecn')" name="ingress_ecn" depend="qdisc_advanced == 1" initial="ECN" :options="ecn" :description="$t('Explicit congestion notification (ECN) status on inbound packets (ingress)')"></uci-option-list>
                <uci-option-list :label="$t('Egress ecn')" name="egress_ecn" depend="qdisc_advanced == 1" initial="NOECN" :options="ecn" :description="$t('Explicit congestion notification (ECN) status on outbound packets (egress)')"></uci-option-list>
                <uci-option-switch :label="$t('Really advanced')"  depend="qdisc_advanced == 1" name="qdisc_really_really_advanced" :description="$t('Show and Use Dangerous Configuration. Dangerous options will only be used as long as this box is checked')"></uci-option-switch>
                <uci-option-input :label="$t('Hard limit on ingress')" name="ilimit" depend="qdisc_really_really_advanced == 1" :description="$t('Hard limit on ingress queues; leave empty for default.')" :rules="{type:'uinteger', min: 0}"></uci-option-input>
                <uci-option-input :label="$t('Hard limit on egress')" name="elimit" depend="qdisc_really_really_advanced == 1" :description="$t('Hard limit on egress queues; leave empty for default')" :rules="{type:'uinteger', min: 0}"></uci-option-input>
                <uci-option-input :label="$t('Latency target for ingress')" name="itarget" depend="qdisc_really_really_advanced == 1" :description="$t('Latency target for ingress, e.g 5ms [units: s, ms, or  us]; leave empty for automatic selection, put in the word default for the qdisc\'s default.')"></uci-option-input>
                <uci-option-input :label="$t('Latency target for egress')" name="etarget" depend="qdisc_really_really_advanced == 1" :description="$t('Latency target for egress, e.g. 5ms [units: s, ms, or  us]; leave empty for automatic selection, put in the word default for the qdisc\'s default.')"></uci-option-input>
                <uci-option-input :label="$t('iqdisc options')" name="iqdisc_opts" depend="qdisc_really_really_advanced == 1" :description="$t('Advanced option string to pass to the ingress queueing disciplines; no error checking, use very carefully.')"></uci-option-input>
                <uci-option-input :label="$t('eqdisc options')" name="eqdisc_opts" depend="qdisc_really_really_advanced == 1" :description="$t('Advanced option string to pass to the egress queueing disciplines; no error checking, use very carefully.')"></uci-option-input>
            </uci-tab>
            <uci-tab :title="$t('Link Layer Adaptation')" name="tab_linklayer">
                <uci-option-list :label="$t('Link layer')" name="linklayer" :options="linklayer" :description="$t('Which link layer to account')"></uci-option-list>
                <uci-option-input :label="$t('Over head')" name="overhead" :rules="{type:'integer', min:-1500}" depend="linklayer == 'ethernet' || linklayer == 'atm'" :description="$t('Per Packet Overhead (byte)')"></uci-option-input>
                <uci-option-switch :label="$t('Advanced Linklayer Options')" name="linklayer_advanced" depend="linklayer == 'ethernet' || linklayer == 'atm'" :description="$t('Show Advanced Linklayer Options, (only needed if MTU > 1500). Advanced options will only be used as long as this box is checked.')"></uci-option-switch>
                <uci-option-input :label="$t('tcMTU')" name="tcMTU" depend="linklayer_advanced == 1" initial=2047 :rules="{type:'uinteger', min:0}" :description="$t('Maximal Size for size and rate calculations, tcMTU (byte); needs to be >= interface MTU + overhead')"></uci-option-input>
                <uci-option-input :label="$t('tcTSIZE')" name="tcTSIZE" depend="linklayer_advanced == 1" initial=128 :rules="{type:'uinteger', min:0}" :description="$t('Number of entries in size/rate tables, TSIZE; for ATM choose TSIZE = (tcMTU + 1) / 16')"></uci-option-input>
                <uci-option-input :label="$t('tcMPU')" name="tcMPU" depend="linklayer_advanced == 1" initial=0 :rules="{type:'uinteger', min:0}" :description="$t('Minimal packet size, MPU (byte); needs to be > 0 for ethernet size tables')"></uci-option-input>
                <uci-option-list :label="$t('adaptation mechanism')" name="linklayer_adaptation_mechanism" depend="linklayer_advanced == 1" initial="default" :options="adaptation_mechanism" :description="$t('Which linklayer adaptation mechanism to use; for testing only')"></uci-option-list>
            </uci-tab>
        </uci-section>
    </uci-form>
</template>
<script>
export default {
  data() {
    return {
      devices:[],
      qdisk:[],
      scripts:[],
      squash_dscp:[
        [1, this.$t('SQUASH')],
        [0, this.$t('DO NOT SQUASH')]
      ],
      squash_ingress:[
        [1, this.$t('Ignore')],
        [0, this.$t('Allow')]
      ],
      ecn:[
        'ECN',
        'NOECN'
      ],
      linklayer:[
        ['none', this.$t('default')],
        ['ethernet', this.$t('Ethernet with overhead: select for e.g. VDSL2.')],
        ['atm', this.$t('ATM: select for e.g. ADSL1, ADSL2, ADSL2+.')]
      ],
      adaptation_mechanism:[
        ['default', this.$t('default')],
        ['cake', this.$t('cake')],
        ['htb_private', this.$t('htb private')],
        ['tc_stab', this.$t('tc stab')]
      ]
    }
  },
  methods:{
    qdisk_change() {
      this.$ubus.call('oui.system', 'read_dir', {path:'/tmp/run/sqm/available_qdiscs'}).then(r => {
        let files = [];
        r.files.forEach(f => {
          files.push(
            f.name
          )
        });
        this.qdisk = files;
      });
    },
    script_change() {
      this.$ubus.call('oui.system', 'read_dir', {path:'/usr/lib/sqm'}).then(r => {
        let files = [];
        r.files.forEach(f => {
          if (f.name.endsWith('.qos'))
            files.push(f.name);
        });
        this.scripts = files;
      });
    }
  },
  created() {
    this.$network.load().then(() => {
      this.devices = this.$network.getDevices().map(d => d.name);
    });
  }
}
</script>