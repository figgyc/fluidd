<template>
  <v-card-text>
    <v-row>
      <v-col cols="12" lg="8">
        <span v-if="meshes.length === 0">No existing bed meshes found.</span>
        <v-simple-table v-if="meshes.length > 0">
          <thead>
            <tr>
              <th>Name</th>
              <th>&nbsp;</th>
              <th>Variance</th>
              <th>&nbsp;</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="mesh in meshes" :key="mesh.profile_name">
              <td class="grey--text text--lighten-1 text-body-1">
                {{ mesh.profile_name }}
              </td>
              <td>
                <v-chip
                  v-if="mesh.active"
                  color="secondary"
                  small
                  block>
                  active
                </v-chip>
              </td>
              <td class="grey--text text--lighten-1 text-body-1"><span v-if="mesh.active">{{ variance.toFixed(4) }}</span></td>
              <td>
                <v-btn
                  @click="loadProfile(mesh.profile_name)"
                  :disabled="mesh.active || hasWaits || printerPrinting || printerBusy"
                  color="secondary"
                  small>
                  Load Profile
                </v-btn>
              </td>
            </tr>
            <tr>
              <td>&nbsp;</td>
              <td>&nbsp;</td>
              <td>&nbsp;</td>
              <td>
                <v-btn
                  @click="clearMesh()"
                  :disabled="!meshLoaded"
                  color="secondary"
                  small>
                  Clear Profile
                </v-btn>
              </td>
            </tr>
          </tbody>
        </v-simple-table>
      </v-col>
      <v-col cols="12" lg="4">
        <v-tooltip right>
          <template v-slot:activator="{ on, attrs }">
            <v-btn
              v-bind="attrs"
              v-on="on"
              block
              class="mb-2"
              color="secondary"
              :loading="hasWait(waits.onMeshCalibrate)"
              :disabled="hasWaits || printerPrinting || printerBusy"
              @click="calibrate()">
              Calibrate
            </v-btn>
          </template>
          <span>Begins a new calibration, saving as profile 'default'</span>
        </v-tooltip>
        <v-btn
          @click="sendGcode('G28', waits.onHomeAll)"
          block
          class="mb-2"
          :loading="hasWait(waits.onHomeAll)"
          :disabled="hasWaits || printerPrinting || printerBusy"
          :color="(!allHomed) ? 'warning' : 'secondary'">
            <v-icon small class="mr-1">$home</v-icon> All
        </v-btn>
        <v-btn
          v-if="!printerPrinting && printerSupportsQgl"
          @click="sendGcode('QUAD_GANTRY_LEVEL', waits.onQGL)"
          :loading="hasWait(waits.onQGL)"
          :disabled="hasWaits || printerPrinting || printerBusy"
          block
          class="mb-2"
          color="secondary">
            QGL
        </v-btn>
        <v-tooltip right>
          <template v-slot:activator="{ on, attrs }">
            <v-btn
              v-bind="attrs"
              v-on="on"
              block
              class="mb-2"
              color="warning"
              :disabled="!meshLoaded || hasWaits || printerPrinting || printerBusy"
              @click="openSaveDialog()">
              Save Config As...
            </v-btn>
          </template>
          <span>Commits calibrated profile to printer.cfg. This WILL restart your printer.</span>
        </v-tooltip>
      </v-col>
    </v-row>

    <dialog-input
      title="Save config as..."
      v-model="saveDialog.open"
      @save="saveToConfig(saveDialog)"
      :max-width="450">
      <!-- <p>
        By default, save overrites the currently loaded profile ({{ currentMesh.profile_name }}).
      </p> -->
      <v-text-field
        filled
        required
        class="mb-4"
        :rules="[value => !!value || 'Required.']"
        hide-details="auto"
        label="Profile Name"
        v-model="saveDialog.profileName">
      </v-text-field>

      <v-checkbox
        :label="`Remove ${currentMesh.profile_name} profile`"
        hide-details="auto"
        class="mb-4"
        v-model="saveDialog.removeDefault"
        :disabled="saveDialog.profileName === currentMesh.profile_name"
        ></v-checkbox>
      <p>
        If saving as something other than {{ currentMesh.profile_name }}, you can choose to
        also remove the {{ currentMesh.profile_name }} profile.
      </p>
    </dialog-input>

    <Plotly v-if="meshLoaded" :data="data" :layout="layout" :display-mode-bar="false"></Plotly>
  </v-card-text>
</template>

<script lang="ts">
import { Component, Mixins, Watch } from 'vue-property-decorator'
import UtilsMixin from '@/mixins/utils'
import { BedMesh } from '@/store/socket/types'
import { MeshData } from '@/types'
import { defaultPlotLayout, Waits } from '@/globals'
import DialogInput from '@/components/dialogs/dialogInput.vue'
import { SaveMeshDialog } from '@/types/dialogs'
import { Plotly } from '@cadriel/vue-plotly'

@Component({
  components: {
    Plotly,
    DialogInput
  }
})
export default class BedMeshWidget extends Mixins(UtilsMixin) {
  waits = Waits
  layout = defaultPlotLayout
  variance = 0
  saveDialog: SaveMeshDialog = {
    open: false,
    profileName: 'default',
    removeDefault: false
  }

  @Watch('saveDialog', { deep: true })
  onSaveDialogChange (val: SaveMeshDialog) {
    if (val.profileName === this.currentMesh.profile_name) {
      this.saveDialog.removeDefault = false
    }
  }

  get meshes (): BedMesh[] {
    return this.$store.getters['socket/getBedMeshes']
  }

  get currentMesh (): BedMesh {
    return this.$store.state.socket.printer.bed_mesh
  }

  get meshLoaded (): boolean {
    return (this.currentMesh && this.currentMesh.profile_name !== '')
  }

  get hasDefault (): boolean {
    return (this.meshes.findIndex(mesh => mesh.profile_name === 'default') > -1)
  }

  @Watch('currentMesh', { deep: true })
  onCurrentMeshChange (val: BedMesh) {
    if (val) {
      this.applyMesh(val)
    }
  }

  calibrate () {
    this.sendGcode('BED_MESH_CALIBRATE', Waits.onMeshCalibrate)
  }

  clearMesh () {
    this.sendGcode('BED_MESH_CLEAR')
  }

  loadProfile (name: string) {
    this.sendGcode('BED_MESH_PROFILE LOAD=' + name)
  }

  saveToConfig (config: SaveMeshDialog) {
    if (config.profileName !== this.currentMesh.profile_name) {
      this.sendGcode('BED_MESH_PROFILE SAVE=' + config.profileName)
    }
    if (config.removeDefault) {
      this.sendGcode('BED_MESH_PROFILE REMOVE=' + this.currentMesh.profile_name)
    }
    this.sendGcode('SAVE_CONFIG')
  }

  mounted () {
    if (this.currentMesh) {
      this.applyMesh(this.currentMesh)
    }
  }

  openSaveDialog () {
    this.saveDialog.profileName = this.currentMesh.profile_name
    this.saveDialog.open = true
  }

  data: MeshData[] = [
    {
      x: [],
      y: [],
      z: [],
      type: 'surface',
      intensity: [0, 0.05, 0.1],
      cmin: -0.1,
      cmax: 0.1,
      showscale: true,
      // autocolorscale: true,
      // colorscale: 'Rainbow',
      colorscale: [[0, 'rebeccapurple'], [0.1, 'rebeccapurple'], [0.2, 'blue'], [0.5, 'green'], [0.8, 'yellow'], [0.9, 'red'], [1, 'red']],
      colorbar: {
        tickfont: {
          color: '#999'
        }
      }
    }
  ]

  /**
   * Applies given mesh data to the plotly graph.
   */
  applyMesh (mesh: BedMesh) {
    // const mesh: BedMesh = this.currentMesh

    if (mesh) {
      // Reset any existing mesh.
      this.data[0].x = []
      this.data[0].y = []
      this.data[0].z = []

      // Define the range for our X and Y axes.
      // If possible, use the actual range from config - otherwise revert to klippers given min and max's.
      const stepperX = this.$store.state.socket.printer.configfile.config.stepper_x
      const stepperY = this.$store.state.socket.printer.configfile.config.stepper_y
      // const stepperZ = this.$store.state.socket.printer.configfile.config.stepper_z

      const layout = defaultPlotLayout

      if (
        mesh.mesh_matrix &&
        mesh.mesh_matrix.length &&
        mesh.mesh_matrix[0].length &&
        mesh.mesh_max &&
        mesh.mesh_min
      ) {
        if (stepperX && stepperY) {
          layout.scene.xaxis.range = [stepperX.position_min, stepperX.position_max]
          layout.scene.yaxis.range = [stepperY.position_min, stepperY.position_max]
        } else {
          layout.scene.xaxis.range = [mesh.mesh_min[0], mesh.mesh_max[0]]
          layout.scene.yaxis.range = [mesh.mesh_min[1], mesh.mesh_max[1]]
        }

        const xCount = mesh.mesh_matrix[0].length
        const yCount = mesh.mesh_matrix.length
        this.data[0].z = mesh.mesh_matrix
        let zMin = Math.min(...mesh.mesh_matrix.map(row => Math.min(...row)))
        let zMax = Math.max(...mesh.mesh_matrix.map(row => Math.max(...row)))
        this.variance = Math.abs(zMin - zMax)

        const xStep = (mesh.mesh_max[0] - mesh.mesh_min[0]) / (xCount - 1)
        for (let i = 0; i < xCount; i++) {
          this.data[0].x.push(mesh.mesh_min[0] + xStep * i)
        }

        const yStep = (mesh.mesh_max[1] - mesh.mesh_min[1]) / (yCount - 1)
        for (let i = 0; i < yCount; i++) {
          this.data[0].y.push(mesh.mesh_min[1] + yStep * i)
        }

        // If the zmin and zmax don't exceed -1 and 1 respectively - then
        // set the min and maxes to -1 and 1 - effectively ensuring we don't
        // have a min max smaller than -1 and 1.
        if (zMin > -1) zMin = -1
        if (zMax < 1) zMax = 1

        layout.scene.zaxis.range = [zMin, zMax]

        this.layout = layout
      }
    }
  }
}
</script>

<style lang="scss" scoped>
  .theme--dark.v-data-table > .v-data-table__wrapper > table > tbody > tr:hover:not(.v-data-table__expanded__content):not(.v-data-table__empty-wrapper) {
    background-color: transparent;
  }
</style>
