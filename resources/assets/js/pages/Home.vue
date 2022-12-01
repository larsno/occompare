<template>
    <div class="container py-3">
        <div class="row">
            <div class="col-12 text-center">
                <div class="form">
                    <div class="form-group row">
                        <div class="col-md-5">
                            <label>Occupation 1</label>
                            <select-occupation v-model="occupation_1"></select-occupation>
                        </div>
                        <div class="col-md-2">
                            <button class="btn btn-danger btn-block mt-4" @click.prevent="compare" :disabled="!occupation_1 || !occupation_2 || loading">
                                <template v-if="loading">
                                    <i class="fa fa-refresh fa-spin"></i>
                                </template>
                                <template v-else>
                                    Compare
                                </template>
                            </button>
                        </div>
                        <div class="col-md-5">
                            <label>Occupation 2</label>
                            <select-occupation v-model="occupation_2"></select-occupation>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <div class="row">
            <template v-if="match !== null && !loading">
            <div class="col-12 text-center">
                <h1>Skill Match {{ match }}%</h1>
            </div>
            <div  class="col-12" v-if="skills && !loading">
                <table class="table table-striped table-sm">
                    <tbody>
                        <tr v-for="skill in skills">
                            <td class="col-md-4 text-center">{{skill.value_1}}</td>
                            <td class="table-success text-center">{{skill.label}}</td>
                            <td class="col-md-4 text-center">{{skill.value_2}}</td>
                        </tr>
                    </tbody>
                </table>
            </div>
            </template>
            <template v-else-if="match === null && !loading">
                <div class="col-12 text-center">
                    Please select two Occupations from above and click Compare
                </div>
            </template>
            <template v-else-if="loading">
                <div class="col-12 text-center">
                    Please wait...
                </div>
            </template>
        </div>
    </div>
</template>

<script>
    import SelectOccupation from '../components/form-controls/SelectOccupation';

    const mergeSkillsTo = (field) => (skillMap, { label, value, description })  => {
        if (skillMap[label]) skillMap[label][field] = +value || 0;
        else {
            skillMap[label] = { label, description };
            skillMap[label][field] = +value || 0;
        }
        return skillMap;
    }
    const sortByMaxValue = ({ max_value: maxA }, { max_value: maxB }) => maxA > maxB ? -1 : (maxA < maxB ? 1 : 0);
    function includeMaxValue (skill) {
        skill.max_value = Math.max(skill.value_1 || 0, skill.value_2 || 0);
        return skill;
    }
    function mergeSkills(occupation_1, occupation_2) {
        let skillMap = {};
        skillMap = occupation_1.reduce(mergeSkillsTo("value_1"), skillMap);
        skillMap = occupation_2.reduce(mergeSkillsTo("value_2"), skillMap);
        const skills = Object.values(skillMap).map(includeMaxValue).sort(sortByMaxValue);
        return skills;
    }
    function calculateMatch(skills) {
        const sum = skills.reduce((sum, { value_1 = 0, value_2 = 0, max_value = 0 }) => {
            // The difference in a skill's importance should be scaled by the size of the importance
            sum.ofDifference += (value_1 === 0 || value_2 === 0 ? 100 : Math.abs(value_1 - value_2)) * max_value;
            sum.ofMaxValue += max_value;
            return sum;
        }, { ofDifference: 0, ofMaxValue: 0 });
        return Math.round(100 - sum.ofDifference / sum.ofMaxValue);
    }

    export default {
        name: 'home-page',
        components: {
            SelectOccupation
        },
        data() {
            return {
                loading: false,
                occupation_1: null,
                occupation_2: null,
                match: null,
                skills: [],
            }
        },
        methods: {
            compare() {
                this.loading = true;
                this.axios.post('/api/compare', {
                    occupation_1: this.occupation_1,
                    occupation_2: this.occupation_2
                }).then((response) => {
                    this.loading = false;
                    this.skills = mergeSkills(response.data.occupation_1, response.data.occupation_2);
                    this.match = calculateMatch(this.skills); // response.data.match;
                }).catch((e) => {
                    this.loading = false;
                });
            }
        },
        watch: {
            occupation_1() {
                this.match = null;
                this.skills = [];
            },
            occupation_2() {
                this.match = null;
                this.skills = [];
            },
        },
    }
</script>

<style lang="scss" scoped>
    .form-group {
        label {
            font-size: 0.8rem;
            text-align: left;
            display: block;
            margin-bottom: 0.2rem
        }
    }
</style>