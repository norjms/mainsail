<template>
    <div v-if="exists" class="printer-setup">
        <v-divider class="mb-4" />
        <div class="d-flex align-center mb-3">
            <v-icon small class="mr-2">{{ mdiTune }}</v-icon>
            <span class="subtitle-2">{{ $t('Panels.MachineSettingsPanel.PrinterSetup.Headline') }}</span>
            <v-spacer />
            <v-chip v-if="watchtowerEnabled" x-small :color="watchtowerColor" outlined :title="watchtowerMessage">
                Watchtower: {{ watchtowerStatus }}
            </v-chip>
        </div>

        <v-row dense>
            <v-col :class="colClass">
                <v-select
                    :value="nozzleSize"
                    :items="nozzleSizeItems"
                    :label="$t('Panels.MachineSettingsPanel.PrinterSetup.NozzleSize')"
                    :disabled="printerIsPrinting"
                    outlined
                    dense
                    hide-details
                    @change="setNozzleSize" />
            </v-col>
            <v-col :class="colClass">
                <v-select
                    :value="nozzleType"
                    :items="options.nozzle_types || []"
                    :label="$t('Panels.MachineSettingsPanel.PrinterSetup.NozzleType')"
                    outlined
                    dense
                    hide-details
                    @change="setNozzleType" />
            </v-col>
        </v-row>

        <v-row dense>
            <v-col :class="colClass">
                <v-select
                    :value="plate"
                    :items="options.plates || []"
                    :label="$t('Panels.MachineSettingsPanel.PrinterSetup.BuildPlate')"
                    :disabled="printerIsPrinting"
                    outlined
                    dense
                    hide-details
                    @change="setPlate" />
            </v-col>
            <v-col :class="colClass" class="d-flex align-center">
                <v-text-field
                    :value="plateOffsetText"
                    :label="$t('Panels.MachineSettingsPanel.PrinterSetup.PlateZOffset')"
                    outlined
                    dense
                    hide-details
                    readonly
                    suffix="mm" />
                <v-btn
                    icon
                    small
                    class="ml-1"
                    :disabled="!plate || printerIsPrinting"
                    :title="$t('Panels.MachineSettingsPanel.PrinterSetup.SaveOffsetHint')"
                    @click="savePlateOffset">
                    <v-icon small>{{ mdiContentSave }}</v-icon>
                </v-btn>
            </v-col>
        </v-row>

        <v-row dense>
            <v-col :class="colClass3">
                <v-select
                    :value="material"
                    :items="options.materials || []"
                    :label="$t('Panels.MachineSettingsPanel.PrinterSetup.Material')"
                    outlined
                    dense
                    hide-details
                    @change="setMaterial" />
            </v-col>
            <v-col :class="colClass3">
                <v-select
                    :value="colorHex"
                    :items="colorItems"
                    item-text="name"
                    item-value="hex"
                    :label="$t('Panels.MachineSettingsPanel.PrinterSetup.Color')"
                    outlined
                    dense
                    hide-details
                    @change="setColor">
                    <template #selection="{ item }">
                        <span class="printer-setup__swatch mr-2" :style="{ backgroundColor: '#' + item.hex }" />
                        {{ item.name }}
                    </template>
                    <template #item="{ item }">
                        <span class="printer-setup__swatch mr-2" :style="{ backgroundColor: '#' + item.hex }" />
                        {{ item.name }}
                    </template>
                </v-select>
            </v-col>
            <v-col :class="colClass3">
                <v-select
                    :value="brand"
                    :items="options.brands || []"
                    :label="$t('Panels.MachineSettingsPanel.PrinterSetup.Brand')"
                    outlined
                    dense
                    hide-details
                    @change="setBrand" />
            </v-col>
        </v-row>

        <v-alert v-if="restartNeeded" dense text type="warning" class="mt-3 mb-0">
            <div class="d-flex align-center">
                <span>
                    {{
                        $t('Panels.MachineSettingsPanel.PrinterSetup.RestartNeeded', {
                            configured: formatSize(configuredNozzle),
                        })
                    }}
                </span>
                <v-spacer />
                <v-btn x-small color="warning" :disabled="printerIsPrinting" @click="restartKlipper">
                    {{ $t('Panels.MachineSettingsPanel.PrinterSetup.Restart') }}
                </v-btn>
            </div>
        </v-alert>

        <div v-if="watchtowerEnabled" class="d-flex align-center mt-2">
            <span class="caption text--secondary text-truncate">{{ watchtowerMessage }}</span>
            <v-spacer />
            <v-btn x-small text @click="sync">{{ $t('Panels.MachineSettingsPanel.PrinterSetup.Sync') }}</v-btn>
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Mixins, Prop } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import { mdiContentSave, mdiTune } from '@mdi/js'

interface PrinterSetupColor {
    name: string
    hex: string
}

interface PrinterSetupOptions {
    nozzle_sizes?: number[]
    nozzle_types?: string[]
    plates?: string[]
    materials?: string[]
    colors?: PrinterSetupColor[]
    brands?: string[]
}

interface PrinterSetupStatus {
    nozzle_size?: number | null
    nozzle_type?: string | null
    configured_nozzle?: number | null
    restart_needed?: boolean
    plate?: string | null
    plate_offset?: number | null
    options?: PrinterSetupOptions
    filament_material?: string | null
    filament_color?: string | null
    filament_color_hex?: string | null
    filament_brand?: string | null
    watchtower_enabled?: boolean
    watchtower_status?: string
    watchtower_message?: string
}

// Dropdowns for the printer_setup Klipper extra (nozzle, build plate, loaded
// filament). Shown only when the printer has a [printer_setup] section.
@Component
export default class MachineSettingsPrinterSetup extends Mixins(BaseMixin) {
    mdiContentSave = mdiContentSave
    mdiTune = mdiTune

    @Prop({ type: Boolean, default: false }) readonly narrow!: boolean

    get status(): PrinterSetupStatus | null {
        return this.$store.state.printer?.printer_setup ?? null
    }

    get exists(): boolean {
        return this.status !== null
    }

    get options(): PrinterSetupOptions {
        return this.status?.options ?? {}
    }

    get colClass(): string {
        return this.narrow ? 'col-12' : 'col-6'
    }

    get colClass3(): string {
        return this.narrow ? 'col-12' : 'col-4'
    }

    get nozzleSize(): number | null {
        return this.status?.nozzle_size ?? null
    }

    get nozzleType(): string | null {
        return this.status?.nozzle_type ?? null
    }

    get configuredNozzle(): number | null {
        return this.status?.configured_nozzle ?? null
    }

    get restartNeeded(): boolean {
        return this.status?.restart_needed ?? false
    }

    get plate(): string | null {
        return this.status?.plate ?? null
    }

    get plateOffsetText(): string {
        const offset = this.status?.plate_offset
        if (offset === null || offset === undefined) return ''
        return Number(offset).toFixed(3)
    }

    get material(): string | null {
        return this.status?.filament_material ?? null
    }

    get colorHex(): string | null {
        return this.status?.filament_color_hex ?? null
    }

    get brand(): string | null {
        return this.status?.filament_brand ?? null
    }

    get colorItems(): PrinterSetupColor[] {
        const items = [...(this.options.colors ?? [])]
        // Keep a custom color (set from the console) selectable and visible.
        if (this.colorHex && !items.some((item) => item.hex === this.colorHex)) {
            items.push({ name: this.status?.filament_color ?? '#' + this.colorHex, hex: this.colorHex })
        }
        return items
    }

    get nozzleSizeItems(): { text: string; value: number }[] {
        return (this.options.nozzle_sizes ?? []).map((size) => ({ text: this.formatSize(size), value: size }))
    }

    get watchtowerEnabled(): boolean {
        return this.status?.watchtower_enabled ?? false
    }

    get watchtowerStatus(): string {
        return this.status?.watchtower_status ?? 'never'
    }

    get watchtowerMessage(): string {
        return this.status?.watchtower_message ?? ''
    }

    get watchtowerColor(): string {
        if (this.watchtowerStatus === 'ok') return 'success'
        if (this.watchtowerStatus === 'error') return 'error'
        return 'grey'
    }

    formatSize(size: number | null): string {
        if (size === null || size === undefined) return ''
        return Number(size).toFixed(2) + ' mm'
    }

    quote(value: string): string {
        return '"' + value.replace(/"/g, '') + '"'
    }

    send(gcode: string): void {
        this.$store.dispatch('server/addEvent', { message: gcode, type: 'command' })
        this.$socket.emit('printer.gcode.script', { script: gcode })
    }

    setNozzleSize(size: number): void {
        const type = this.nozzleType ? ' TYPE=' + this.quote(this.nozzleType) : ''
        this.send(`PRINTER_SETUP_NOZZLE SIZE=${size}${type} QUIET=1`)
    }

    setNozzleType(type: string): void {
        this.send(`PRINTER_SETUP_NOZZLE TYPE=${this.quote(type)} QUIET=1`)
    }

    setPlate(name: string): void {
        this.send(`PRINTER_SETUP_PLATE NAME=${this.quote(name)} QUIET=1`)
    }

    savePlateOffset(): void {
        this.send('PRINTER_SETUP_PLATE_OFFSET')
    }

    setMaterial(material: string): void {
        this.send(`PRINTER_SETUP_FILAMENT MATERIAL=${material} QUIET=1`)
    }

    setColor(hex: string): void {
        this.send(`PRINTER_SETUP_FILAMENT COLOR=${hex} QUIET=1`)
    }

    setBrand(brand: string): void {
        this.send(`PRINTER_SETUP_FILAMENT BRAND=${this.quote(brand)} QUIET=1`)
    }

    restartKlipper(): void {
        this.send('PRINTER_SETUP_RESTART')
    }

    sync(): void {
        this.send('PRINTER_SETUP_SYNC')
    }
}
</script>

<style scoped>
.printer-setup__swatch {
    display: inline-block;
    width: 14px;
    height: 14px;
    border-radius: 3px;
    border: 1px solid rgba(255, 255, 255, 0.4);
    vertical-align: middle;
}
</style>
