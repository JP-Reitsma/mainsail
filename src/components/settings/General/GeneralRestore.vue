<template>
    <div>
        <input ref="uploadBackupFile" type="file" :accept="['.json']" class="d-none" @change="uploadRestore" />
        <div class="d-flex align-center ga-2">
            <v-btn
                small
                :loading="loadings.includes('restoreUploadButton')"
                @click="restoreDb">
                From...
            </v-btn>

            <v-btn
                small
                :loading="loadings.includes('restorePrinterButton')"
                @click="restoreFromPrinterConfig">
                From Printer Config
            </v-btn>
        </div>
        <v-dialog :value="showDialog" persistent :width="360">
            <panel
                :title="$t('Settings.GeneralTab.Restore')"
                card-class="mainsail-restore-dialog"
                :margin-bottom="false"
                :icon="mdiHelpCircle">
                <template #buttons>
                    <v-btn icon tile @click="closeDialog">
                        <v-icon>{{ mdiCloseThick }}</v-icon>
                    </v-btn>
                </template>
                <v-card-text>
                    <v-row>
                        <v-col>
                            <p class="mb-0">{{ $t('Settings.GeneralTab.RestoreDialog') }}</p>
                        </v-col>
                    </v-row>
                    <v-row>
                        <checkbox-list
                            :options="restoreableNamespaces"
                            select-all
                            @update:selectedCheckboxes="onSelectRestoreCheckboxes" />
                    </v-row>
                    <v-row>
                        <v-col class="text-center">
                            <v-btn color="red" :loading="loadings.includes('restoreMainsail')" @click="restoreDbAction">
                                {{ $t('Settings.GeneralTab.Restore') }}
                            </v-btn>
                        </v-col>
                    </v-row>
                </v-card-text>
            </panel>
        </v-dialog>
    </div>
</template>

<script lang="ts">
import Component from 'vue-class-component'
import { Mixins, Ref } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import SettingsRow from '@/components/settings/SettingsRow.vue'
import Panel from '@/components/ui/Panel.vue'
import { mdiCloseThick, mdiHelpCircle } from '@mdi/js'
import CheckboxList from '@/components/inputs/CheckboxList.vue'
import { TranslateResult } from 'vue-i18n'
import SettingsGeneralDatabase from '@/components/mixins/settingsGeneralDatabase'
import SettingsGeneralTabBackupDatabase from '@/components/settings/General/GeneralBackup.vue'
import Vue from 'vue'

@Component({
    components: { SettingsGeneralTabBackupDatabase, Panel, SettingsRow, CheckboxList },
})
export default class SettingsGeneralTabRestoreDatabase extends Mixins(BaseMixin, SettingsGeneralDatabase) {
    mdiHelpCircle = mdiHelpCircle
    mdiCloseThick = mdiCloseThick

    @Ref() readonly uploadBackupFile!: HTMLInputElement

    showDialog = false
    restoreableNamespaces: { value: string; label: string | TranslateResult }[] = []
    restoreCheckboxes: string[] = []
    restoreObjects: Record<string, unknown> = {}

    onSelectRestoreCheckboxes(backupCheckboxes: string[]) {
        this.restoreCheckboxes = backupCheckboxes
    }

    async restoreDb() {
        this.uploadBackupFile.click()
    }

    async restoreFromPrinterConfig() {
        const url =
            this.$store.getters['socket/getUrl'] +
            '/server/files/config/.mainsail/backup-mainsail.json?time=' +
            Date.now()

        try {
            const restoreObjects = await fetch(url).then((r) => {
                if (r.status !== 200)
                    throw new Error('Backup file not found')

                return r.json()
            })

            this.prepareRestoreObjects(restoreObjects)
        } catch (e) {
            Vue.$toast.error('Printer backup not found')
        }
    }

    uploadRestore() {
        const backup = this.uploadBackupFile?.files?.[0]
        if (!backup) {
            window.console.error('No json uploaded')
            return
        }

        const reader = new FileReader()
        reader.readAsText(backup, 'UTF-8')
        reader.onload = (evt) => {
            this.restoreableNamespaces = []
            try {
                const restoreObjects = JSON.parse(evt?.target?.result + '')
                this.prepareRestoreObjects(restoreObjects)
            } catch (e) {
                Vue.$toast.error(this.$t('Settings.GeneralTab.CannotReadJson').toString())
            }
        }
        reader.onerror = (evt) => {
            window.console.error(evt)
        }

        // empty input file field
        this.uploadBackupFile.value = ''
    }

    prepareRestoreObjects(restoreObjects: any) {
        this.restoreableNamespaces = []
        this.restoreObjects = restoreObjects

        const keys = Object.keys(this.restoreObjects)

        this.restoreableNamespaces = keys.map((key) => {
            const namespace = this.availableKeys.find((namespace) => namespace.value === key)
            if (namespace) return namespace

            return { value: key, label: key }
        })

        this.restoreableNamespaces =
            this.restoreableNamespaces.sort(this.sortNamespaces)

        this.openDialog()
    }

    openDialog() {
        this.showDialog = true
    }

    closeDialog() {
        this.showDialog = false
    }

    restoreDbAction() {
        this.$store.dispatch('socket/addLoading', 'restoreDbAction')

        this.$store.dispatch('gui/restoreMoonrakerDB', {
            dbCheckboxes: this.restoreCheckboxes,
            restoreObjects: this.restoreObjects,
        })
    }
}
</script>
