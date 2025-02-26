<script>
import StudyItem from "./StudyItem.vue"
import ResourceButtonGroup from "./ResourceButtonGroup.vue"

import { mapState, mapGetters } from "vuex"
import { baseOe2Url } from "../globalConfigurations"
import { translateDicomTag } from "../locales/i18n"
import dateHelpers from "../helpers/date-helpers"
import $ from "jquery"
import { endOfMonth, endOfYear, startOfMonth, startOfYear, subMonths, subDays, startOfWeek, endOfWeek, subYears } from 'date-fns';
import api from "../orthancApi";
import { ref } from 'vue';

document._allowedFilters = ["StudyDate", "StudyTime", "AccessionNumber", "PatientID", "PatientName", "PatientBirthDate", "StudyInstanceUID", "StudyID", "StudyDescription", "ModalitiesInStudy", "labels"]

document._studyColumns = {
    "StudyDate": {
        "width": "7%"
    },
    "AccessionNumber": {
        "width": "11%",
        "placeholder": "1234"
    },
    "PatientID": {
        "width": "11%",
        "placeholder": "1234"
    },
    "PatientName": {
        "width": "15%",
        "placeholder": "John^Doe"
    },
    "PatientBirthDate": {
        "width": "7%"
    },
    "StudyDescription": {
        "width": "25%",
        "placeholder": "Chest"
    },
    "modalities": {
        "width": "6%"
    },
    "seriesCount": {
        "width": "4%"
    },
    "undefined": {
        "width": "10%"
    }
};

export default {
    props: [],
    emits: ['deletedStudy'],
    data() {
        return {
            filterStudyDate: '',
            filterStudyDateForDatePicker: '',
            filterPatientBirthDate: '',
            filterPatientBirthDateForDatePicker: '',
            filterModalities: {},
            filterGenericTags : {},
            oldFilterGenericTags : {},
            filterLabels: [],
            allModalities: true,
            noneModalities: false,
            updatingFilterUi: false,
            initializingModalityFilter: false,
            searchTimerHandler: {},
            columns: document._studyColumns,
            datePickerPresetRanges: document._datePickerPresetRanges,
            allSelected: false,
            isPartialSelected: false,
            latestStudiesIds: [],
            shouldStopLoadingLatestStudies: false,
            isLoadingLatestStudies: false,
            isDisplayingLatestStudies: false,
        };
    },
    computed: {
        ...mapState({
            uiOptions: state => state.configuration.uiOptions,
            isConfigurationLoaded: state => state.configuration.loaded,
            studiesIds: state => state.studies.studiesIds,
            selectedStudiesIds: state => state.studies.selectedStudiesIds,
            isSearching: state => state.studies.isSearching,
            statistics: state => state.studies.statistics
        }),
        ...mapGetters([
            'studies/isFilterEmpty', // -> this['studies/isFilterEmpty']
        ]),
        notShowingAllResults() {
            if (this.studiesIds.length >= this.statistics.CountStudies) {
                return false;
            }
            return this.studiesIds.length == this.uiOptions.MaxStudiesDisplayed; // in this case, the result has been limited
        },
        isDarkMode() {
            // hack to switch the theme: get the value from our custom css
            let bootstrapTheme = document.documentElement.getAttribute("data-bs-theme"); // for production
            bootstrapTheme = getComputedStyle(document.documentElement).getPropertyValue('--bootstrap-theme');  // for dev
            console.log("DatePicker color mode is ", bootstrapTheme);
            return bootstrapTheme == "dark";
        },
        isSearchAsYouTypeEnabled() {
            return this.uiOptions.StudyListSearchMode == "search-as-you-type";
        },
        isSearchButtonEnabled() {
            return this.uiOptions.StudyListSearchMode == "search-button";
        },
        showEmptyStudyListIfNoSearch() {
            return this.uiOptions.StudyListContentIfNoSearch == "empty";
        },
        showLastReceivedIfNoSearch() {
            return this.uiOptions.StudyListContentIfNoSearch == "most-recents";
        },
        isStudyListEmpty() {
            return this.studiesIds.length == 0;
        },
        datePickerFormat() {
            return this.uiOptions.DateFormat;
        }
    },
    watch: {
        '$route': async function () { // the watch is used when, e.g, clicking on the back button
            this.updateFilterFromRoute(this.$route.query);
        },
        isConfigurationLoaded(newValue, oldValue) {
            // this is called when opening the page (with a filter or not)
            // console.log("StudyList: Configuration has been loaded, updating study filter: ", this.$route.params.filters);
            this.initModalityFilter();
            for (const tag of this.uiOptions.StudyListColumns) {
            if (['StudyDate', 'PatientBirthDate', 'modalities', 'seriesCount'].indexOf(tag) == -1) {
                this.filterGenericTags[tag] = '';
            }
        }
            this.updateFilterFromRoute(this.$route.query);
        },
        filterLabels: {
            handler(newValue, oldValue) {
                if (!this.updatingFilterUi && !this.initializingModalityFilter) {
                    //    console.log("StudyList: filterModalities watcher", newValue, oldValue);
                    this.updateFilter('labels', this.filterLabels, null);
                }
            },
            deep: true
        },
        filterModalities: {
            handler(newValue, oldValue) {
                if (!this.updatingFilterUi && !this.initializingModalityFilter) {
                    if (this.isSearchAsYouTypeEnabled) {
                        //    console.log("StudyList: filterModalities watcher", newValue, oldValue);
                        this.updateFilter('ModalitiesInStudy', this.getModalityFilter(), null);
                    } else {
                        this.getModalityFilter(); // to update all/none status
                    }
                }
            },
            deep: true
        },
        filterGenericTags: {
            handler(newValue, oldValue) {
                // oldValue is the same as newValue for deep watchers
                for (const [k, v] of Object.entries(this.filterGenericTags)) {
                    let oldValue = null;
                    if (k in this.oldFilterGenericTags) {
                        oldValue = this.oldFilterGenericTags[k]
                    }
                    this.updateFilter(k, v, oldValue);
                    this.oldFilterGenericTags[k] = v;
                }
            },
            deep: true
        },
        filterStudyDate(newValue, oldValue) {
            // console.log("watch filterStudyDate", newValue);
            this.updateFilter('StudyDate', newValue, oldValue);
        },
        filterStudyDateForDatePicker(newValue, oldValue) {
            let dicomNewValue = dateHelpers.dicomDateFromDatePicker(newValue);
            if (dicomNewValue == null) {
                dicomNewValue = "";
            }
            // console.log("watch filterStudyDateForDatePicker", newValue, dicomNewValue);
            this.filterStudyDate = dicomNewValue;
        },
        filterPatientBirthDate(newValue, oldValue) {
            this.updateFilter('PatientBirthDate', newValue, oldValue);
        },
        filterPatientBirthDateForDatePicker(newValue, oldValue) {
            let dicomNewValue = dateHelpers.dicomDateFromDatePicker(newValue);
            if (dicomNewValue == null) {
                dicomNewValue = "";
            }
            // console.log("watch filterPatientBirthDateForDatePicker", newValue, dicomNewValue);
            this.filterPatientBirthDate = dicomNewValue;
        },
        selectedStudiesIds: {
            handler(oldValue, newValue) {
                this.updateSelectAll();
            },
            deep: true
        },
    },
    async created() {
        this.messageBus.on('language-changed', this.translateDatePicker);
        this.messageBus.on('filter-label-changed', this.updateLabelsFilter);
    },
    async mounted() {
        this.updateSelectAll();
    },
    methods: {
        updateSelectAll() {
            if (this.selectedStudiesIds.length == 0) {
                this.allSelected = false;
                this.isPartialSelected = false;
            } else if (this.selectedStudiesIds.length == this.studiesIds.length) {
                this.allSelected = true;
                this.isPartialSelected = false;
            } else {
                this.allSelected = '';
                this.isPartialSelected = true;
            }
        },
        clickSelectAll() {
            if (this.allSelected == '' || !this.allSelected) { // this is the value before the click
                this.$store.dispatch('studies/selectAllStudies', {isSelected: true});
                this.messageBus.emit('selected-all');
            } else {
                this.$store.dispatch('studies/selectAllStudies', {isSelected: false});
                this.messageBus.emit('unselected-all')
            }
        },
        translateDatePicker(languageKey) {
            for (let i in document._datePickerPresetRanges) {
                document._datePickerPresetRanges[i].label = this.$t(document._datePickerPresetRanges[i].tLabel);
            }
        },
        columnTitle(tagName) {
            if (tagName == "seriesCount") {
                return this.$i18n.t('series_count_header');
            } else if (tagName == "modalities") {
                return translateDicomTag(this.$i18n.t, this.$i18n.te, "ModalitiesInStudy");
            } else {
                return translateDicomTag(this.$i18n.t, this.$i18n.te, tagName);
            }
        },
        columnTooltip(tagName) {
            return this.columnTitle(tagName);
        },
        columnWidth(tagName) {
            if (tagName in this.columns) {
                return this.columns[tagName].width;
            } else {
                return this.columns[undefined].width;
            }
        },
        clearModalityFilter() {
            // console.log("StudyList: clearModalityFilter", this.updatingFilterUi);
            for (const modality of this.uiOptions.ModalitiesFilter) {
                this.filterModalities[modality] = true;
            }
        },
        updateLabelsFilter(label) {
            this.filterLabels = [label];
        },
        initModalityFilter() {
            // console.log("StudyList: initModalityFilter", this.updatingFilterUi);
            this.initializingModalityFilter = true;
            this.filterModalities = {};
            for (const modality of this.uiOptions.ModalitiesFilter) {
                this.filterModalities[modality] = true;
            }
            this.initializingModalityFilter = false;
        },
        getModalityFilter() {
            if (this.filterModalities === undefined) {
                return "";
            }

            let modalityFilter = "";
            let allSelected = true;
            let selected = [];

            for (const [key, value] of Object.entries(this.filterModalities)) {
                allSelected &= value;
                if (value) {
                    selected.push(key);
                }
            }
            if (allSelected) {
                this.allModalities = true;
                this.noneModalities = false;
                return "";
            } else if (selected.length == 0) {
                this.allModalities = false;
                this.noneModalities = true;
                return "NONE"; // something that will not match !
            } else {
                this.allModalities = false;
                this.noneModalities = false;
                return selected.join('\\');
            }
        },
        updateFilter(dicomTagName, newValue, oldValue) {

            if (this.updatingFilterUi) {
                return;
            }

            // console.log("StudyList: updateFilter", this.updatingFilterUi);

            if (dicomTagName == "ModalitiesInStudy" && oldValue == null) { // not text: e.g. modalities in study -> update directly
                this._updateFilter(dicomTagName, newValue);
                return;
            }

            if (dicomTagName == "labels" && newValue != oldValue) {
                this._updateLabelsFilter(newValue);
            }

            if (!this.isSearchAsYouTypeEnabled) {
                return;
            }

            if (newValue.length >= this.uiOptions.StudyListSearchAsYouTypeMinChars) {
                // calls updateFilter only after a delay without any key pressed and if there are enough characters entered
                if (this.searchTimerHandler[dicomTagName]) {
                    clearTimeout(this.searchTimerHandler[dicomTagName]);
                }
                this.searchTimerHandler[dicomTagName] = setTimeout(() => { this._updateFilter(dicomTagName, newValue) }, this.uiOptions.StudyListSearchAsYouTypeDelay);
            } else if (oldValue && newValue.length < oldValue.length && oldValue.length >= this.uiOptions.StudyListSearchAsYouTypeMinChars) { // when deleting filter
                this.searchTimerHandler[dicomTagName] = setTimeout(() => { this._updateFilter(dicomTagName, "") }, this.uiOptions.StudyListSearchAsYouTypeDelay);
            }
        },
        clipFilter(dicomTagName, value) {
            if (this.isFilterLongEnough(dicomTagName, value)) {
                return value;
            } else {
                return "";
            }
        },
        getMinimalFilterLength(dicomTagName) {
            if (["AccessionNumber", "PatientName", "PatientID", "StudyDescription"].indexOf(dicomTagName) != -1) {
                if (this.isSearchAsYouTypeEnabled) {
                    return this.uiOptions.StudyListSearchAsYouTypeMinChars;
                }
            } else if (["PatientBirthDate", "StudyDate"].indexOf(dicomTagName) != -1) {
                return 8;
            }
            return 0;
        },
        isFilterLongEnough(dicomTagName, value) {
            return value.length >= this.getMinimalFilterLength(dicomTagName);
        },
        getFilterClass(dicomTagName) {
            const value = this.getFilterValue(dicomTagName)
            if (value != null && value.length > 0 && !this.isFilterLongEnough(dicomTagName, value)) {
                return "is-invalid-filter";
            }
            return "";
        },
        hasFilter(tagName) {
            return ['seriesCount'].indexOf(tagName) == -1;
        },
        getFilterPlaceholder(tagName) {
            if (tagName in this.columns && this.columns[tagName].placeholder) {
                return this.columns[tagName].placeholder;
            } else {
                return "search-text";
            }
        },
        getFilterValue(dicomTagName) {
            if (!this.isConfigurationLoaded) {
                return null;
            }
            if (dicomTagName == "StudyDate") {
                return this.filterStudyDate;
            } else if (dicomTagName == "PatientBirthDate") {
                return this.filterPatientBirthDate;
            } else if (dicomTagName == "ModalitiesInStudy") {
                console.error("getFilterValue ModalitiesInStudy");
            } else {
                return this.filterGenericTags[dicomTagName];
            }
        },
        _updateLabelsFilter(labels) {
            this.$store.dispatch('studies/updateLabelsFilterNoReload', { labels: labels });
            this.updateUrl();
            this.reloadStudyList();
        },
        _updateFilter(dicomTagName, value) {
            this.searchTimerHandler[dicomTagName] = null;
            this.$store.dispatch('studies/updateFilterNoReload', { dicomTagName: dicomTagName, value: value });
            this.updateUrl();
            this.reloadStudyList();
        },
        async updateFilterFromRoute(filters) {
            //console.log("StudyList: updateFilterFromRoute", this.updatingFilterUi, filters);

            this.updatingFilterUi = true;
            await this.$store.dispatch('studies/clearFilterNoReload');
            var keyValueFilters = {};

            for (const [filterKey, filterValue] of Object.entries(filters)) {
                if (filterKey == "labels") {
                    const labels = filterValue.split(",");
                    keyValueFilters[filterKey] = labels;
                    await this.$store.dispatch('studies/updateLabelsFilterNoReload', { labels: labels });
                } else if (filterKey[0] === filterKey[0].toUpperCase()) {  // DicomTags starts with a capital letter
                    keyValueFilters[filterKey] = filterValue;
                    await this.$store.dispatch('studies/updateFilterNoReload', { dicomTagName: filterKey, value: filterValue });
                }
            }

            await this.updateFilterForm(keyValueFilters);
            await this.reloadStudyList();

            this.updatingFilterUi = false;
        },
        updateFilterForm(filters) {
            // console.log("StudyList: updateFilterForm", this.updatingFilterUi);
            this.emptyFilterForm();

            for (const [key, value] of Object.entries(filters)) {
                if (key == "labels") {
                    this.filterLabels = value;
                } else if (key == "StudyDate") {
                    this.filterStudyDate = value;
                    this.filterStudyDateForDatePicker = dateHelpers.parseDateForDatePicker(value);
                } else if (key == "PatientBirthDate") {
                    this.filterPatientBirthDate = value;
                    this.filterPatientBirthDateForDatePicker = dateHelpers.parseDateForDatePicker(value);
                } else if (key == "ModalitiesInStudy") {
                    const modalities = value.split('\\');
                    if (modalities.length > 0) {
                        let allModalitiesInFilter = true;
                        let noneModalitiesInFilter = true;
                        for (const modality of this.uiOptions.ModalitiesFilter) {
                            const isInFilter = modalities.indexOf(modality) != -1
                            this.filterModalities[modality] = isInFilter;
                            allModalitiesInFilter &= isInFilter;
                            noneModalitiesInFilter &= !isInFilter;
                        }
                        this.allModalities = allModalitiesInFilter;
                        this.noneModalities = noneModalitiesInFilter;
                    }
                } else {
                    this.filterGenericTags[key] = value;
                }
            }
        },
        emptyFilterForm() {
            // console.log("StudyList: emptyFilterForm", this.updatingFilterUi);
            this.filterStudyDate = '';
            this.filterStudyDateForDatePicker = null;
            this.filterPatientBirthDate = '';
            this.filterPatientBirthDateForDatePicker = null;
            this.filterGenericTags = {};
            for (const tag of this.uiOptions.StudyListColumns) {
                if (['StudyDate', 'PatientBirthDate', 'modalities', 'seriesCount'].indexOf(tag) == -1) {
                    this.filterGenericTags[tag] = '';
                }
            }
            this.filterLabels = [];
            this.clearModalityFilter();
        },
        isFilteringOnlyOnLabels() {
            let hasGenericTagFilter = false;
            for (const tag of this.uiOptions.StudyListColumns) {
                if (['StudyDate', 'PatientBirthDate', 'modalities', 'seriesCount'].indexOf(tag) == -1) {
                    if (this.filterGenericTags[tag] && this.filterGenericTags[tag] != '') {
                        hasGenericTagFilter = true;
                    }
                }
            }
            return this.filterStudyDate == '' && this.filterPatientBirthDate == '' && !hasGenericTagFilter && this.filterLabels.length > 0;
        },
        async search() {
            if (this.isSearching) {
                await this.$store.dispatch('studies/cancelSearch');
            } else {
                await this.$store.dispatch('studies/clearFilterNoReload');
                for (const tag of this.uiOptions.StudyListColumns) {
                    if (['modalities', 'seriesCount'].indexOf(tag) == -1) {
                        await this.$store.dispatch('studies/updateFilterNoReload', { dicomTagName: tag, value: this.getFilterValue(tag) });    
                    }
                }
                await this.reloadStudyList();
                this.updateUrl();
            }
        },
        async clearFilters() {
            // console.log("StudyList: clearFilters", this.updatingFilterUi);
            await this.clearFiltersUi();
            await this.$store.dispatch('studies/clearFilterNoReload');

            this.reloadStudyList();
        },
        async clearFiltersUi() {
            // console.log("StudyList: clearFiltersUi IN");
            this.updatingFilterUi = true;

            this.emptyFilterForm();
            this.updateUrl();

            this.updatingFilterUi = false;
            // console.log("StudyList: clearFiltersUi OUT");
        },
        async toggleModalityFilter(ev) {
            // only for all/none, other values are binded with v-model !
            const modality = ev.srcElement.getAttribute("data-value");
            let newValue = true;
            if (modality == "all") {
                newValue = true;
            } else if (modality == "none") {
                newValue = false;
            }

            for (const [key, value] of Object.entries(this.filterModalities)) {
                this.filterModalities[key] = newValue;
            }

            this.getModalityFilter(); // to update the state of "all"/"none"
        },
        modalityFilterClicked(ev) {  // prevent closing the drop-down at every click
            ev.stopPropagation();
        },
        closeModalityFilter(ev) {
            // simulate a click on the dropdown toggle (TODO: fix error in console)
            $("#dropdown-modalities-button").click();
            ev.preventDefault();
            ev.stopPropagation();
        },
        updateUrl() {
            let activeFilters = [];
            if (this.clipFilter("StudyDate", this.filterStudyDate)) {
                activeFilters.push('StudyDate=' + this.filterStudyDate);
            }
            if (this.clipFilter("PatientBirthDate", this.filterPatientBirthDate)) {
                activeFilters.push('PatientBirthDate=' + this.filterPatientBirthDate);
            }
            if (this.getModalityFilter()) {
                activeFilters.push('ModalitiesInStudy=' + this.getModalityFilter());
            }
            for (const [k, v] of Object.entries(this.filterGenericTags)) {
                if (this.clipFilter(k, v)) {
                    activeFilters.push(k + '=' + v);
                }
            }
            if (this.filterLabels.length > 0) {
                activeFilters.push('labels=' + this.filterLabels.join(","))
            }

            let newUrl = "";
            if (activeFilters.length > 0) {
                newUrl = "/filtered-studies?" + activeFilters.join('&');
            }

            this.$router.replace(newUrl);
        },
        async reloadStudyList() {
            // if we are displaying most recent studies and there is only a label filter -> continue to show the list of most recent studies (filtered by label)
            const shouldShowMostRecentsWithLabel = this.uiOptions.StudyListContentIfNoSearch == "most-recents" && this.isFilteringOnlyOnLabels();

            if (this['studies/isFilterEmpty'] || shouldShowMostRecentsWithLabel) {
                await this.$store.dispatch('studies/clearStudies');
                if (this.uiOptions.StudyListContentIfNoSearch == "empty") {
                    return;
                } else if (this.uiOptions.StudyListContentIfNoSearch == "most-recents") {
                    if (this.isLoadingLatestStudies) {
                        // if currently loading, stop it
                        this.shouldStopLoadingLatestStudies = true;
                        this.isLoadingLatestStudies = false;
                        this.isDisplayingLatestStudies = true;
                    }
                    // restart loading 
                    const lastChangeId = await api.getLastChangeId();
                
                    await this.$store.dispatch('studies/clearStudies');
                    this.latestStudiesIds = new Set();
                    this.shouldStopLoadingLatestStudies = false;
                    this.isLoadingLatestStudies = true;
                    this.isDisplayingLatestStudies = false;

                    this.loadStudiesFromChange(Math.max(0, lastChangeId - 1000), 1000);
                }
            } else {
                this.shouldStopLoadingLatestStudies = true;
                this.isLoadingLatestStudies = false;
                this.isDisplayingLatestStudies = false;
                // await this.$store.dispatch('studies/updateLabelsFilterNoReload', { labels: this.filterLabels });
                await this.$store.dispatch('studies/reloadFilteredStudies');
            }
        },
        async loadStudiesFromChange(fromChangeId, limit) {
            let changes = await api.getChanges(fromChangeId, limit);
            for (let change of changes["Changes"].reverse()) {
                // Take the first event we find -> we see last uploaded data immediately (NewStudy but no StableStudy).  
                // An updated study that has received a new series is visible as well (its NewStudy might be too old but the StableStudy brings it back on top of the list)
                if ((change["ChangeType"] == "NewStudy" || change["ChangeType"] == "StableStudy") && !this.latestStudiesIds.has(change["ID"])) {
                    if (this.shouldStopLoadingLatestStudies) {
                        return;
                    }
                    //console.log(change);
                    try {
                        const study = await api.getStudy(change["ID"]);
                        if (this.filterLabels.length == 0 || this.filterLabels.filter(l => study["Labels"].includes(l)).length > 0) {
                            this.$store.dispatch('studies/addStudy', { studyId: change["ID"], study: study, reloadStats: false });
                        }

                        this.latestStudiesIds.add(change["ID"]);
                        if (this.latestStudiesIds.size == this.uiOptions.MaxStudiesDisplayed) {
                            this.isLoadingLatestStudies = false;
                            this.isDisplayingLatestStudies = true;
                            return;
                        }
                    } catch (err) {
                        console.warn("Unable to load study - not authorized ?");
                    }
                }
            }
            if (!this.shouldStopLoadingLatestStudies) {
                if (fromChangeId > 0 && this.latestStudiesIds.size < this.statistics.CountStudies) {
                    setTimeout(() => {this.loadStudiesFromChange(Math.max(0, Math.max(0, fromChangeId - 1000)), 1000)}, 1);
                } else {
                    this.isLoadingLatestStudies = false;
                    this.isDisplayingLatestStudies = true;
                }
            } else {
                this.isLoadingLatestStudies = false;
                this.isDisplayingLatestStudies = true;
            }
        },
        onDeletedStudy(studyId) {
            this.$store.dispatch('studies/deleteStudy', { studyId: studyId });
        },
    },
    components: { StudyItem, ResourceButtonGroup }
}
</script>


<template>
    <div>
        <table class="table table-responsive table-sm study-table table-borderless">
            <thead class="study-table-header">
                <th width="2%" scope="col" ></th>
                <th v-for="columnTag in uiOptions.StudyListColumns" :key="columnTag" data-bs-toggle="tooltip"
                    v-bind:title="columnTooltip(columnTag)" v-bind:width="columnWidth(columnTag)"
                    class="study-table-title">{{
                        columnTitle(columnTag)
                    }}</th>
            </thead>
            <thead class="study-table-filters" v-on:keyup.enter="search">
                <th scope="col">
                    <button @click="clearFilters" type="button" class="form-control study-list-filter btn filter-button"
                        data-bs-toggle="tooltip" title="Clear filter">
                        <i class="fa-regular fa-circle-xmark"></i>
                    </button>
                </th>
                <th v-for="columnTag in uiOptions.StudyListColumns" :key="columnTag">
                    <div v-if="columnTag == 'StudyDate'">
                        <Datepicker v-if="columnTag == 'StudyDate'" v-model="filterStudyDateForDatePicker"
                            :enable-time-picker="false" range :preset-dates="datePickerPresetRanges" :format="datePickerFormat"
                            :preview-format="datePickerFormat" text-input arrow-navigation :highlight-week-days="[0, 6]" :dark="isDarkMode">
                            <template #yearly="{ label, range, presetDate }">
                                <span @click="presetDate(range)">{{ label }}</span>
                            </template>
                        </Datepicker>
                    </div>
                    <div v-else-if="columnTag == 'modalities'" class="dropdown">
                        <button type="button" class="btn btn-default btn-sm filter-button dropdown-toggle"
                            data-bs-toggle="dropdown" id="dropdown-modalities-button" aria-expanded="false"><span
                                class="fa fa-list"></span>&nbsp;<span class="caret"></span></button>
                        <ul class="dropdown-menu" aria-labelledby="dropdown-modalities-button"
                            @click="modalityFilterClicked" id="modality-filter-dropdown">
                            <li><label class="dropdown-item"><input type="checkbox" data-value="all"
                                        @click="toggleModalityFilter" v-model="allModalities" />&nbsp;{{
                                            $t('all_modalities') }}</label></li>
                            <li><label class="dropdown-item"><input type="checkbox" data-value="none"
                                        @click="toggleModalityFilter" v-model="noneModalities" />&nbsp;{{
                                            $t('no_modalities') }}</label></li>
                            <li>
                                <hr class="dropdown-divider">
                            </li>
                            <li v-for="modality in uiOptions.ModalitiesFilter" :key="modality">
                                <label class="dropdown-item"><input type="checkbox" v-bind:data-value="modality"
                                        v-model="filterModalities[modality]" />&nbsp;{{ modality }}</label>
                            </li>
                            <li><button class="btn btn-primary mx-5" @click="closeModalityFilter">{{ $t('close') }}</button></li>
                        </ul>
                    </div>
                    <div v-else-if="columnTag == 'PatientBirthDate'">
                        <Datepicker v-model="filterPatientBirthDateForDatePicker"
                            :enable-time-picker="false" range :format="datePickerFormat" :preview-format="datePickerFormat" text-input
                            arrow-navigation :highlight-week-days="[0, 6]" :dark="isDarkMode">
                        </Datepicker>
                    </div>
                    <input v-else-if="hasFilter(columnTag)" type="text" class="form-control study-list-filter"
                        v-model="this.filterGenericTags[columnTag]" v-bind:placeholder="getFilterPlaceholder(columnTag)"
                        v-bind:class="getFilterClass(columnTag)" />
                </th>
            </thead>
            <thead class="study-table-actions">
                <th width="2%" scope="col">
                    <div class="form-check" style="margin-left: 1.9rem">
                        <input class="form-check-input" type="checkbox" v-model="allSelected"
                            :indeterminate="isPartialSelected" @click="clickSelectAll">
                    </div>
                </th>
                <th width="98%" colspan="10" scope="col">
                    <div class="container">
                        <div class="row g-1">
                            <div class="col-4 study-list-bulk-buttons">
                                <ResourceButtonGroup :resourceLevel="'bulk'">
                                </ResourceButtonGroup>
                            </div>
                            <div class="col-6">
                                <div v-if="!isSearching && isLoadingLatestStudies" class="alert alert-secondary study-list-alert" role="alert">
                                    <span v-if="isLoadingLatestStudies" class="spinner-border spinner-border-sm alert-icon" role="status"
                                        aria-hidden="true"></span>{{
                                            $t('loading_latest_studies') }}
                                </div>
                                <div v-else-if="!isSearching && isDisplayingLatestStudies" class="alert alert-secondary study-list-alert" role="alert">
                                    <i class="bi bi-exclamation-triangle-fill alert-icon"></i>{{
                                            $t('displaying_latest_studies') }}
                                </div>
                                <div v-else-if="!isSearching && notShowingAllResults" class="alert alert-danger study-list-alert"
                                    role="alert">
                                    <i class="bi bi-exclamation-triangle-fill alert-icon"></i> {{ $t('not_showing_all_results') }} ! !
                                </div>
                                <div v-else-if="!isSearching && showEmptyStudyListIfNoSearch && this['studies/isFilterEmpty']"
                                    class="alert alert-warning study-list-alert" role="alert">
                                    <i class="bi bi-exclamation-triangle-fill alert-icon"></i> {{ $t('enter_search') }}
                                </div>
                                <div v-else-if="!isSearching && isStudyListEmpty"
                                    class="alert alert-warning study-list-alert" role="alert">
                                    <i class="bi bi-exclamation-triangle-fill alert-icon"></i> {{ $t('no_result_found') }}
                                </div>
                                <div v-else-if="isSearching" class="alert alert-secondary study-list-alert" role="alert">
                                    <span v-if="isSearching" class="spinner-border spinner-border-sm alert-icon" role="status"
                                        aria-hidden="true"></span>{{
                                            $t('searching') }}
                                </div>
                            </div>
                            <div class="col-2">
                                <button @click="search" v-if="isSearchButtonEnabled" type="submit"
                                    class="form-control study-list-filter btn filter-button btn-secondary search-button"
                                    data-bs-toggle="tooltip"
                                    :class="{ 'is-searching': isSearching, 'is-not-searching': !isSearching }"
                                    title="Search">
                                    <i v-if="!isSearching" class="fa-solid fa-magnifying-glass"></i>
                                    <span v-if="isSearching" class="spinner-border spinner-border-sm" role="status"
                                        aria-hidden="true"></span>
                                </button>
                            </div>
                        </div>
                    </div>
                </th>
            </thead>
            <StudyItem v-for="studyId in studiesIds" :key="studyId" :studyId="studyId"
                @deletedStudy="onDeletedStudy">
            </StudyItem>

        </table>
    </div>
</template>

<style>
:root {
    --filter-margin: 5px;
    --filter-padding: 2px;
}

/* styles are in common.css because they are shared between the StudyList and the RemoteStudyList */

</style>