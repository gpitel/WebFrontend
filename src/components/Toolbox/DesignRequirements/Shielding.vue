<script setup>
import { useMasStore } from '../../../stores/mas'
import Button from 'primevue/button'
import InputText from 'primevue/inputtext'
import Select from 'primevue/select'
</script>

<script>
export default {
    props: {
        dataTestLabel: {
            type: String,
            default: '',
        },
    },
    data() {
        const masStore = useMasStore();
        return {
            masStore,
        }
    },
    computed: {
        shields() {
            return this.masStore.mas.inputs.designRequirements.shielding || [];
        },
        windingNames() {
            return (this.masStore.mas.magnetic.coil.functionalDescription || [])
                .map((winding, index) => winding.name || 'Winding ' + (index + 1));
        },
        // Every unordered winding pair; at the requirements stage the coil is not wound
        // yet, so a shield declared here applies to every interface of its pair
        pairOptions() {
            // windings are referenced by name, as everywhere else in MAS; renames are
            // propagated into shield references by renameWinding()
            const names = this.windingNames;
            const options = [];
            for (let i = 0; i < names.length; i++) {
                for (let j = i + 1; j < names.length; j++) {
                    options.push({
                        value: names[i] + '|' + names[j],
                        label: names[i] + ' ↔ ' + names[j],
                    });
                }
            }
            return options;
        },
    },
    methods: {
        pairValue(requirement) {
            const pair = requirement.betweenWindings || [];
            if (pair.length != 2) {
                return null;
            }
            // normalize to the option ordering (lower winding index first)
            const names = this.windingNames;
            if (names.indexOf(pair[0]) > names.indexOf(pair[1])) {
                return pair[1] + '|' + pair[0];
            }
            return pair[0] + '|' + pair[1];
        },
        addShield() {
            const designRequirements = this.masStore.mas.inputs.designRequirements;
            const firstPair = this.pairOptions[0];
            const betweenWindings = firstPair ? firstPair.value.split('|') : [];
            designRequirements.shielding = [
                ...(designRequirements.shielding || []),
                {
                    name: 'Shield ' + ((designRequirements.shielding || []).length + 1),
                    betweenWindings,
                },
            ];
        },
        removeShield(requirement) {
            const designRequirements = this.masStore.mas.inputs.designRequirements;
            designRequirements.shielding = (designRequirements.shielding || []).filter((existing) => existing !== requirement);
        },
        shieldNameChanged(requirement, value) {
            if (value != '') {
                requirement.name = value;
            }
        },
        shieldPairChanged(requirement, value) {
            if (value == null) {
                return;
            }
            requirement.betweenWindings = value.split('|');
        },
    }
}
</script>

<template>
    <div class="shielding-req">
        <div class="shielding-req-header">
            <label
                :style="$styleStore.designRequirements.inputTitleFontSize"
                class="shielding-req-title"
            >
                Shielding
            </label>
            <Button
                class="shielding-req-add-btn"
                severity="secondary"
                outlined
                :data-cy="dataTestLabel + '-AddShield'"
                :disabled="pairOptions.length == 0"
                @click="addShield"
            >
                <i class="pi pi-plus"></i>
                <span>Add shield</span>
            </Button>
        </div>
        <label v-if="pairOptions.length == 0" class="shielding-req-hint">
            Shields need at least two windings.
        </label>
        <label v-else-if="shields.length == 0" class="shielding-req-hint">
            Electrostatic screens placed between two windings. A shield declared here is
            applied at every interface between its windings; refine placement, thickness
            and termination in the Magnetic Builder.
        </label>
        <div v-for="(requirement, requirementIndex) in shields" :key="requirementIndex" class="shielding-req-row">
            <InputText
                class="shielding-req-name"
                :data-cy="dataTestLabel + '-ShieldName-' + requirementIndex"
                :model-value="requirement.name || 'Shield ' + (requirementIndex + 1)"
                @change="shieldNameChanged(requirement, $event.target.value)"
            />
            <Select
                class="shielding-req-pair"
                :data-cy="dataTestLabel + '-ShieldPair-' + requirementIndex"
                :options="pairOptions"
                optionLabel="label"
                optionValue="value"
                :model-value="pairValue(requirement)"
                @update:model-value="shieldPairChanged(requirement, $event)"
            />
            <Button
                class="shielding-req-remove-btn"
                severity="secondary"
                text
                rounded
                :data-cy="dataTestLabel + '-RemoveShield-' + requirementIndex"
                aria-label="Remove shield"
                @click="removeShield(requirement)"
            >
                <i class="pi pi-times"></i>
            </Button>
        </div>
    </div>
</template>

<style scoped>
.shielding-req {
    padding: 0.25rem 0.5rem;
}

.shielding-req-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 0.4rem;
}

.shielding-req-title {
    font-weight: 600;
    margin: 0;
    padding-left: 0.5rem;
}

.shielding-req-add-btn {
    display: inline-flex;
    align-items: center;
    gap: 0.4rem;
    padding: 0.2rem 0.7rem;
    font-size: 0.85rem;
}

.shielding-req-add-btn i {
    font-size: 0.75rem;
}

.shielding-req-hint {
    display: block;
    font-size: 0.85rem;
    opacity: 0.7;
    padding: 0 0.5rem 0.4rem 0.5rem;
}

.shielding-req-row {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    margin-bottom: 0.4rem;
    padding-left: 0.5rem;
}

/* Editable shield name, styled like the winding name inputs: borderless underline */
.shielding-req-name {
    flex: 1 1 30%;
    min-width: 0;
    background: transparent;
    border: 0;
    border-bottom: 1px solid rgba(var(--p-white-rgb), 0.15);
    border-radius: 0;
    box-shadow: none;
    color: inherit;
    font-size: 0.92rem;
    font-weight: 600;
    padding: 0.15rem 0.25rem;
}

.shielding-req-name:focus {
    border-bottom-color: var(--p-primary);
    outline: none;
}

.shielding-req-pair {
    flex: 1 1 55%;
    min-width: 0;
    font-size: 0.88rem;
}

.shielding-req-remove-btn {
    width: 1.8rem;
    height: 1.8rem;
    flex: 0 0 auto;
}
</style>
