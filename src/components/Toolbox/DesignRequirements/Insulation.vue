<script setup>
import { toTitleCase, getMultiplier, combinedStyle, combinedClass } from 'WebSharedComponents/assets/js/utils.js'
import DimensionWithTolerance from 'WebSharedComponents/DataInput/DimensionWithTolerance.vue'
import ElementFromList from 'WebSharedComponents/DataInput/ElementFromList.vue'
import SeveralElementsFromList from 'WebSharedComponents/DataInput/SeveralElementsFromList.vue'
import { minimumMaximumScalePerParameter, defaultInsulationSystemRequirement } from 'WebSharedComponents/assets/js/defaults.js'
import { CTI as Cti, IsolationClass, OvervoltageCategory, PollutionDegree, InsulationStandards, TemperatureClassEnum, InsulationMaterialUse } from 'WebSharedComponents/assets/ts/MAS.ts'
import * as Utils from 'WebSharedComponents/assets/js/utils.js'
import Checkbox from 'primevue/checkbox'
import { useTaskQueueStore } from '../../../stores/taskQueue'
</script>

<script>
export default {
    props: {
        modelValue:{
            type: Object,
            required: true
        },
        defaultValue:{
            type: Object,
            required: true
        },
        dataTestLabel: {
            type: String,
            default: '',
        },
        showTitle:{
            type: Boolean,
            default: true
        },
        standardsToDisable: {
            type: Array,
            default: () => [],
        },
        addButtonStyle: {
            type: Object,
            default: () => ({}),
        },
        removeButtonBgColor: {
            type: String,
            default: "bg-danger",
        },
        valueFontSize: {
            type: [String, Object],
            default: ''
        },
        titleFontSize: {
            type: [String, Object],
            default: ''
        },
        labelBgColor: {
            type: [String, Object],
            default: "bg-transparent",
        },
        valueBgColor: {
            type: [String, Object],
            default: "bg-light",
        },
        textColor: {
            type: [String, Object],
            default: "text-white",
        },
        unitExtraStyleClass:{
            type: String,
            default: ''
        },
    },
    data() {
        const taskQueueStore = useTaskQueueStore();
        return {
            taskQueueStore,
            insulationMaterialNames: [],
        }
    },
    computed: {
        insulationSystem() {
            return this.modelValue['insulation']['insulationSystem'];
        },
        insulationSystemEnabled: {
            get() {
                return this.modelValue['insulation']['insulationSystem'] != null;
            },
            set(enabled) {
                if (enabled) {
                    this.modelValue['insulation']['insulationSystem'] = Utils.deepCopy(defaultInsulationSystemRequirement);
                }
                else {
                    delete this.modelValue['insulation']['insulationSystem'];
                }
                this.$emit('update');
            },
        },
        temperatureClassLabels() {
            const celsiusPerLetter = { Y: 90, A: 105, E: 120, B: 130, F: 155, H: 180, N: 200, R: 220 };
            const out = {};
            Object.values(TemperatureClassEnum).forEach(v => {
                out[v] = v in celsiusPerLetter ? `${v} (${celsiusPerLetter[v]} °C)` : `${v} °C`;
            });
            return out;
        },
        ctiLabels() {
            return { groupI: 'Group I', groupII: 'Group II', groupIIIA: 'Group IIIA', groupIIIB: 'Group IIIB' };
        },
        insulationTypeLabels() {
            const out = {};
            Object.values(IsolationClass).forEach(v => { out[v] = v.charAt(0).toUpperCase() + v.slice(1); });
            return out;
        },
        overvoltageCategoryLabels() {
            // Values are already Roman numerals (II/III/IV) — show as "OVC II" etc.
            const out = {};
            Object.values(OvervoltageCategory).forEach(v => { out[v] = `OVC ${v}`; });
            return out;
        },
        pollutionDegreeLabels() {
            // Values are PD1/PD2/PD3/PD4 — display with space.
            const out = {};
            Object.values(PollutionDegree).forEach(v => { out[v] = v.replace(/^PD/, 'PD '); });
            return out;
        },
    },
    watch: {
    },
    mounted () {
        this.loadInsulationMaterialNames();
    },
    methods: {
        loadInsulationMaterialNames() {
            // Suggestions only — free text stays allowed for off-database
            // materials (varnishes, bobbin stock, ...).
            this.taskQueueStore.getAvailableInsulationMaterials()
                .then((names) => { this.insulationMaterialNames = names; })
                .catch(() => { this.insulationMaterialNames = []; });
        },
        setInsulationSystemText(field, value) {
            if (value == '') {
                delete this.modelValue['insulation']['insulationSystem'][field];
            }
            else {
                this.modelValue['insulation']['insulationSystem'][field] = value;
            }
            this.$emit('update');
        },
        addAllowedMaterial() {
            if (this.modelValue['insulation']['insulationSystem']['allowedMaterials'] == null) {
                this.modelValue['insulation']['insulationSystem']['allowedMaterials'] = [];
            }
            this.modelValue['insulation']['insulationSystem']['allowedMaterials'].push({ name: '' });
            this.$emit('update');
        },
        removeAllowedMaterial(index) {
            this.modelValue['insulation']['insulationSystem']['allowedMaterials'].splice(index, 1);
            if (this.modelValue['insulation']['insulationSystem']['allowedMaterials'].length == 0) {
                delete this.modelValue['insulation']['insulationSystem']['allowedMaterials'];
            }
            this.$emit('update');
        },
        setAllowedMaterialName(index, value) {
            this.modelValue['insulation']['insulationSystem']['allowedMaterials'][index]['name'] = value;
            this.$emit('update');
        },
        cleanEmptyUses(material) {
            // uses has schema minItems 1 — an unchecked-everything row must drop the key
            if (material['uses'] != null && material['uses'].length == 0) {
                delete material['uses'];
            }
            this.$emit('update');
        },
    }
}
</script>

<template>
    <div :data-cy="dataTestLabel + '-container'" class="ins-root">
        <div v-if="showTitle" class="ins-title" :data-cy="dataTestLabel + '-title'">
            <i class="pi pi-shield"></i>
            <span>Insulation</span>
        </div>

        <!-- Section 1: Environment (altitude + main supply voltage) -->
        <div class="ins-section">
            <div class="ins-section-header">
                <i class="pi pi-sun"></i>
                <span>Environment</span>
            </div>
            <div class="ins-grid-2">
                <div class="ins-cell">
                    <DimensionWithTolerance
                        :dataTestLabel="dataTestLabel + '-Altitude'"
                        :allowNegative="true"
                        :min="minimumMaximumScalePerParameter['altitude']['min']"
                        :max="minimumMaximumScalePerParameter['altitude']['max']"
                        :defaultValue="Utils.deepCopy(defaultValue['altitude'])"
                        :halfSize="true"
                        :name="'altitude'"
                        :unit="'m'"
                        v-model="modelValue['insulation']['altitude']"
                        :addButtonStyle="addButtonStyle"
                        :removeButtonBgColor="removeButtonBgColor"
                        :titleFontSize='valueFontSize'
                        :valueFontSize="valueFontSize"
                        :labelBgColor="labelBgColor"
                        :valueBgColor="valueBgColor"
                        :textColor="textColor"
                        :unitExtraStyleClass="unitExtraStyleClass"
                        @update="$emit('update')"
                    />
                </div>
                <div class="ins-cell">
                    <DimensionWithTolerance
                        :dataTestLabel="dataTestLabel + '-MainSupplyVoltage'"
                        :min="minimumMaximumScalePerParameter['voltage']['min']"
                        :max="minimumMaximumScalePerParameter['voltage']['max']"
                        :defaultValue="Utils.deepCopy(defaultValue['mainSupplyVoltage'])"
                        :halfSize="true"
                        :name="'mainSupplyVoltage'"
                        :unit="'V'"
                        v-model="modelValue['insulation']['mainSupplyVoltage']"
                        :addButtonStyle="addButtonStyle"
                        :removeButtonBgColor="removeButtonBgColor"
                        :titleFontSize='valueFontSize'
                        :valueFontSize="valueFontSize"
                        :labelBgColor="labelBgColor"
                        :valueBgColor="valueBgColor"
                        :textColor="textColor"
                        :unitExtraStyleClass="unitExtraStyleClass"
                        @update="$emit('update')"
                    />
                </div>
            </div>
        </div>

        <!-- Section 2: Classification (CTI / IsolationClass / Overvoltage / Pollution) -->
        <div class="ins-section">
            <div class="ins-section-header">
                <i class="pi pi-list"></i>
                <span>Classification</span>
            </div>
            <div class="ins-grid-2">
                <div class="ins-cell">
                    <ElementFromList
                        :dataTestLabel="dataTestLabel + '-Cti'"
                        :name="'cti'"
                        :titleSameRow="true"
                        :justifyContent="true"
                        v-model="modelValue['insulation']"
                        :options="Object.values(Cti)"
                        :optionLabels="ctiLabels"
                        :labelWidthProportionClass="'col-6'"
                        :selectStyleClass="'col-6'"
                        :labelFontSize='valueFontSize'
                        :valueFontSize="valueFontSize"
                        :labelBgColor="labelBgColor"
                        :valueBgColor="valueBgColor"
                        :textColor="textColor"
                        @update="$emit('update')"
                    />
                </div>
                <div class="ins-cell">
                    <ElementFromList
                        :dataTestLabel="dataTestLabel + '-InsulationType'"
                        :name="'insulationType'"
                        :titleSameRow="true"
                        :justifyContent="true"
                        v-model="modelValue['insulation']"
                        :options="Object.values(IsolationClass)"
                        :optionLabels="insulationTypeLabels"
                        :labelWidthProportionClass="'col-6'"
                        :selectStyleClass="'col-6'"
                        :labelFontSize='valueFontSize'
                        :valueFontSize="valueFontSize"
                        :labelBgColor="labelBgColor"
                        :valueBgColor="valueBgColor"
                        :textColor="textColor"
                        @update="$emit('update')"
                    />
                </div>
                <div class="ins-cell">
                    <ElementFromList
                        :dataTestLabel="dataTestLabel + '-OvervoltageCategory'"
                        :name="'overvoltageCategory'"
                        :titleSameRow="true"
                        :justifyContent="true"
                        v-model="modelValue['insulation']"
                        :options="Object.values(OvervoltageCategory)"
                        :optionLabels="overvoltageCategoryLabels"
                        :labelWidthProportionClass="'col-6'"
                        :selectStyleClass="'col-6'"
                        :labelFontSize='valueFontSize'
                        :valueFontSize="valueFontSize"
                        :labelBgColor="labelBgColor"
                        :valueBgColor="valueBgColor"
                        :textColor="textColor"
                        @update="$emit('update')"
                    />
                </div>
                <div class="ins-cell">
                    <ElementFromList
                        :dataTestLabel="dataTestLabel + '-PollutionDegree'"
                        :name="'pollutionDegree'"
                        :titleSameRow="true"
                        :justifyContent="true"
                        v-model="modelValue['insulation']"
                        :options="Object.values(PollutionDegree)"
                        :optionLabels="pollutionDegreeLabels"
                        :labelWidthProportionClass="'col-6'"
                        :selectStyleClass="'col-6'"
                        :labelFontSize='valueFontSize'
                        :valueFontSize="valueFontSize"
                        :labelBgColor="labelBgColor"
                        :valueBgColor="valueBgColor"
                        :textColor="textColor"
                        @update="$emit('update')"
                    />
                </div>
            </div>
        </div>

        <!-- Section 3: Standards -->
        <div class="ins-section">
            <div class="ins-section-header">
                <i class="pi pi-verified"></i>
                <span>Standards</span>
            </div>
            <div class="ins-standards">
                <SeveralElementsFromList
                    :name="'standards'"
                    v-model="modelValue['insulation']"
                    :options="Object.values(InsulationStandards)"
                    :optionsToDisable="standardsToDisable"
                    :labelFontSize='valueFontSize'
                    :valueFontSize="valueFontSize"
                    :labelBgColor="labelBgColor"
                    :valueBgColor="valueBgColor"
                    :textColor="textColor"
                    @update="$emit('update')"
                />
            </div>
        </div>

        <!-- Section 4: Certified insulation system (EIS), e.g. UL 1446 -->
        <div class="ins-section">
            <div class="ins-section-header">
                <i class="pi pi-id-card"></i>
                <span>Insulation System</span>
                <div class="ins-eis-toggle">
                    <Checkbox
                        :data-cy="dataTestLabel + '-InsulationSystem-enable'"
                        inputId="insulation-system-enable"
                        binary
                        v-model="insulationSystemEnabled"
                    />
                    <label for="insulation-system-enable">Required</label>
                </div>
            </div>
            <div v-if="insulationSystem != null" class="ins-eis-body">
                <div class="ins-grid-2">
                    <div class="ins-cell">
                        <ElementFromList
                            :dataTestLabel="dataTestLabel + '-TemperatureClass'"
                            :name="'temperatureClass'"
                            :titleSameRow="true"
                            :justifyContent="true"
                            v-model="modelValue['insulation']['insulationSystem']"
                            :options="Object.values(TemperatureClassEnum)"
                            :optionLabels="temperatureClassLabels"
                            :labelWidthProportionClass="'col-6'"
                            :selectStyleClass="'col-6'"
                            :labelFontSize='valueFontSize'
                            :valueFontSize="valueFontSize"
                            :labelBgColor="labelBgColor"
                            :valueBgColor="valueBgColor"
                            :textColor="textColor"
                            @update="$emit('update')"
                        />
                    </div>
                    <div class="ins-cell ins-eis-text">
                        <label>Name</label>
                        <input
                            :data-cy="dataTestLabel + '-InsulationSystem-name'"
                            type="text"
                            placeholder="e.g. QT-180T"
                            :value="insulationSystem['name']"
                            @change="setInsulationSystemText('name', $event.target.value)"
                        >
                    </div>
                    <div class="ins-cell ins-eis-text">
                        <label>Certificate</label>
                        <input
                            :data-cy="dataTestLabel + '-InsulationSystem-certificate'"
                            type="text"
                            placeholder="e.g. UL E65007"
                            :value="insulationSystem['certificate']"
                            @change="setInsulationSystemText('certificate', $event.target.value)"
                        >
                    </div>
                    <div class="ins-cell ins-eis-text">
                        <label>Certified by</label>
                        <input
                            :data-cy="dataTestLabel + '-InsulationSystem-certifyingOrganization'"
                            type="text"
                            placeholder="e.g. UL"
                            :value="insulationSystem['certifyingOrganization']"
                            @change="setInsulationSystemText('certifyingOrganization', $event.target.value)"
                        >
                    </div>
                </div>

                <div class="ins-eis-materials">
                    <div class="ins-eis-materials-header">
                        <span>Approved materials</span>
                        <button
                            :data-cy="dataTestLabel + '-InsulationSystem-addMaterial'"
                            type="button"
                            class="ins-eis-add"
                            @click="addAllowedMaterial"
                        >
                            <i class="pi pi-plus"></i> Add material
                        </button>
                    </div>
                    <div
                        v-for="(material, materialIndex) in insulationSystem['allowedMaterials']"
                        :key="materialIndex"
                        class="ins-eis-material-row"
                    >
                        <input
                            :data-cy="dataTestLabel + '-InsulationSystem-material-' + materialIndex + '-name'"
                            type="text"
                            class="ins-eis-material-name"
                            placeholder="Material name, e.g. Nomex 410"
                            list="ins-eis-material-names"
                            :value="material['name']"
                            @change="setAllowedMaterialName(materialIndex, $event.target.value)"
                        >
                        <button
                            :data-cy="dataTestLabel + '-InsulationSystem-material-' + materialIndex + '-remove'"
                            type="button"
                            class="ins-eis-remove"
                            @click="removeAllowedMaterial(materialIndex)"
                        >
                            <i class="pi pi-times"></i>
                        </button>
                        <div class="ins-eis-material-uses">
                            <SeveralElementsFromList
                                :dataTestLabel="dataTestLabel + '-InsulationSystem-material-' + materialIndex + '-uses'"
                                :name="'uses'"
                                v-model="insulationSystem['allowedMaterials'][materialIndex]"
                                :options="Object.values(InsulationMaterialUse)"
                                :labelFontSize='valueFontSize'
                                :valueFontSize="valueFontSize"
                                :labelBgColor="labelBgColor"
                                :valueBgColor="valueBgColor"
                                :textColor="textColor"
                                @update="cleanEmptyUses(material)"
                            />
                        </div>
                    </div>
                    <datalist id="ins-eis-material-names">
                        <option v-for="materialName in insulationMaterialNames" :key="materialName" :value="materialName"></option>
                    </datalist>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
.ins-root {
    display: flex;
    flex-direction: column;
    gap: 0.55rem;
    padding: 0.2rem 0.1rem;
}

.ins-title {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    color: var(--p-primary);
    font-weight: 700;
    font-size: 1rem;
    letter-spacing: 0.01em;
    padding: 0.15rem 0.25rem 0.4rem 0.25rem;
}

.ins-title i {
    filter: drop-shadow(0 0 4px rgba(var(--p-primary-rgb), 0.5));
}

/* ============ Sub-section ============ */
.ins-section {
    background: linear-gradient(180deg,
        rgba(var(--p-primary-rgb), 0.05) 0%,
        rgba(var(--p-primary-rgb), 0.015) 100%);
    border: 1px solid rgba(var(--p-primary-rgb), 0.15);
    border-radius: 10px;
    overflow: hidden;
}

.ins-section-header {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    padding: 0.35rem 0.7rem;
    background: rgba(var(--p-primary-rgb), 0.08);
    border-bottom: 1px solid rgba(var(--p-primary-rgb), 0.12);
    color: var(--p-primary);
    font-weight: 600;
    font-size: 0.7rem;
    letter-spacing: 0.05em;
    text-transform: uppercase;
}

.ins-section-header i {
    font-size: 0.78rem;
    filter: drop-shadow(0 0 3px rgba(var(--p-primary-rgb), 0.45));
}

/* ============ Grid layouts ============ */
.ins-grid-2 {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0.4rem 0.85rem;
    padding: 0.55rem 0.7rem;
}

.ins-cell {
    min-width: 0;
}

.ins-standards {
    padding: 0.55rem 0.7rem;
}

/* Tighten the inner rows generated by the child DataInput components */
.ins-cell :deep(.row),
.ins-standards :deep(.row) {
    margin-left: 0 !important;
    margin-right: 0 !important;
}

/* The SeveralElementsFromList component always renders its own "Standards"
   title label — redundant since the section header already says STANDARDS. */
.ins-standards :deep(.several-elements-label) {
    display: none !important;
}
</style>
