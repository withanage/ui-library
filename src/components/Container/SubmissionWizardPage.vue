<script type="text/javascript">
import Page from '@/components/Container/Page.vue';
import ButtonRow from '../ButtonRow/ButtonRow.vue';
import ContributorsListPanel from '../ListPanel/contributors/ContributorsListPanel.vue';
import File from '../File/File.vue';
import Modal from '../Modal/Modal.vue';
import SubmissionFilesListPanel from '../ListPanel/submissionFiles/SubmissionFilesListPanel.vue';
import ajaxError from '@/mixins/ajaxError';
import autosave from '@/mixins/autosave';
import dialog from '@/mixins/dialog';
import localizeMoment from '@/mixins/localizeMoment';
import localizeSubmission from '@/mixins/localizeSubmission';
import localStorage from '@/mixins/localStorage';
import moment from 'moment';

export default {
	extends: Page,
	mixins: [
		ajaxError,
		autosave,
		dialog,
		localizeMoment,
		localizeSubmission,
		localStorage,
	],
	components: {
		ButtonRow,
		ContributorsListPanel,
		File,
		Modal,
		SubmissionFilesListPanel,
	},
	data() {
		return {
			autosavesKeyBase: 'submitAutosaves',
			categories: [],
			currentStepId: 0,
			errors: {},
			isSaving: false,
			isValidating: false,
			lastAutosavedMessage: '',
			publication: {},
			publicationApiUrl: '',
			reconfigurePublicationProps: [],
			reconfigureSubmissionProps: [],
			staleForms: [],
			startedSteps: [],
			steps: [],
			submission: {},
			submissionApiUrl: '',
			submissionSavedUrl: '',
			submissionWizardUrl: '',
			submitApiUrl: '',
			i18nConfirmSubmit: '',
			i18nDiscardChanges: '',
			i18nDisconnected: '',
			i18nLastAutosaved: '',
			i18nPageTitle: '',
			i18nSubmit: '',
			i18nTitleSeparator: '',
			i18nUnableToSave: '',
			i18nUnsavedChanges: '',
			i18nUnsavedChangesMessage: '',
		};
	},
	computed: {
		autosavesKey() {
			return this.autosavesKeyBase + this.submission.id;
		},
		canSubmit() {
			return (
				!this.isAutosaving &&
				!this.isDisconnected &&
				this.isValid &&
				this.isConfirmed
			);
		},
		currentCategoryTitles() {
			return this.publication.categoryIds
				.filter((id) => !!this.categories[id])
				.map((id) => this.categories[id]);
		},
		currentStep() {
			return this.steps.find((step) => step.id === this.currentStepId);
		},
		currentStepIndex() {
			return this.steps.findIndex((step) => step.id === this.currentStepId);
		},
		isConfirmed() {
			const hasUnconfirmedField = this.steps
				.find((step) => step.id === 'review')
				?.sections.find((section) => section.id === 'confirmSubmission')
				?.form.fields.map(
					(field) => field.component !== 'field-options' || field.value
				)
				.includes(false);
			return !hasUnconfirmedField;
		},
		isOnFirstStep() {
			return !this.currentStepIndex;
		},
		isOnLastStep() {
			return this.currentStepIndex === this.steps.length - 1;
		},
		isValid() {
			return Object.keys(this.errors).length === 0;
		},

		/**
		 * The title to show at the top of the page
		 */
		pageTitle() {
			return this.i18nPageTitle.replace('{$step}', this.currentStep.name);
		},
	},
	methods: {
		/**
		 * Add stale data that needs to be autosaved
		 */
		addAutosaves() {
			// Don't autosave if no form data has been changed
			if (!this.staleForms.length) {
				return;
			}

			while (this.staleForms.length) {
				const formId = this.staleForms.splice(0, 1)[0];
				const form = this.$refs.autosaveForms.find((ref) => ref.id === formId);
				if (!form) {
					return;
				}
				this.addAutosave(form.id, form.action, {...form.submitValues});
			}
		},

		/**
		 * Add a step change to the browser history so the
		 * user can use the browser's back button
		 *
		 * @param {Object} step The step to add
		 */
		addHistory(step) {
			window.history.pushState({}, step.name, '#' + step.id);
		},

		/**
		 * A callback function fired when an autosave fails
		 *
		 * If we fail to connect to the server (0) or receive a
		 * 500 server error, we assume that this problem will
		 * eventually be resolved. No action is required.
		 *
		 * If we get a 403 authorization error, we assume that
		 * the user's login or CSRF token has expired and ask
		 * them to reload/login again.
		 *
		 * Otherwise, the problem is probably with the request,
		 * in which case we must delete the invalid autosave
		 * payload and ask the user to reload the page.
		 *
		 * @param {Object} autosave The autosave payload, with the form id at payload.id
		 * @param {jqXHR} xhr jQuery XHR, a superset of XMLHttpRequest
		 * @param {String} status See `error` docs at https://api.jquery.com/jquery.ajax
		 */
		autosaveErrored(autosave, xhr, status) {
			if ([0, 500].includes(xhr.status)) {
				return;
			}

			if (xhr.status !== 403) {
				let autosaves = this.getLocalStorage(this.autosavesKey);
				if (autosaves.length) {
					this.setLocalStorage(
						this.autosavesKey,
						autosaves.filter((payload) => payload.id !== autosave.id)
					);
				}
			}

			this.isDisconnected = true;
			this.ajaxErrorCallback({});
		},

		/**
		 * A callback function fired when the autosave
		 * is complete. Use this to sync the response
		 * data with the submission and publication
		 * objects in data()
		 */
		autosaveSucceeded(autosave, response) {
			if (response.submissionId) {
				this.publication = response;
			} else if (response.dateSubmitted) {
				this.submission = response;
			}
		},

		/**
		 * Get an author's name and affiliation
		 *
		 * @return {String} name, affiliation
		 */
		getAuthorName(author) {
			let affiliation = this.localize(author.affiliation);
			if (affiliation) {
				return [author.fullName, affiliation].join(
					pkp.localeKeys['common.commaListSeparator']
				);
			}
			return author.fullName;
		},

		/**
		 * Get the page's <title> tag for a step
		 *
		 * This gets the title used in the HTML <head> so that the
		 * browser updates the page title as the user navigates the
		 * wizard. It is also used in the browser's history.
		 */
		getPageTitle(step) {
			return document.title
				.split(this.i18nTitleSeparator)
				.map((part, i) => {
					return i ? part : this.i18nPageTitle.replace('{$step}', step.name);
				})
				.join(this.i18nTitleSeparator);
		},

		/**
		 * Go to the next step or submit if this is the last step
		 */
		nextStep() {
			if (this.isOnLastStep) {
				this.submit();
			} else {
				this.openStep(this.steps[1 + this.currentStepIndex].id);
			}
		},

		/**
		 * Show a pop-up when save for later fails
		 */
		openSaveForLaterFailed() {
			this.openDialog({
				name: 'saveForLaterFailed',
				title: this.i18nDisconnected,
				message: this.i18nUnableToSave,
				actions: [
					{
						label: this.__('common.ok'),
						callback: () => this.$modal.hide('saveForLaterFailed'),
					},
				],
			});
		},

		/**
		 * Go to a step in the wizard
		 */
		openStep(stepId) {
			const newStep = this.steps.find((step) => step.id === stepId);
			if (!newStep) {
				return;
			}
			this.currentStepId = stepId;
		},

		/**
		 * Override the #hash behaviour for tabs
		 *
		 * This opens the correct step when the hash is changed
		 */
		openUrlTab() {
			const stepId = window.location.hash.replace('#', '');
			const newStep = this.steps.find((step) => step.id === stepId);
			this.openStep(newStep ? newStep.id : this.steps[0].id);
		},

		/**
		 * Go to the previous step in the wizard
		 */
		previousStep() {
			const previousIndex = this.currentStepIndex - 1;
			if (previousIndex >= 0) {
				this.openStep(this.steps[previousIndex].id);
			}
		},

		/**
		 * When the form to reconfigure a submission is saved
		 */
		reconfigureSubmission(values) {
			const getData = (data, prop) => {
				if (typeof values[prop] !== undefined) {
					data[prop] = values[prop];
				}
				return data;
			};
			const submissionData = this.reconfigureSubmissionProps.reduce(
				getData,
				{}
			);
			const publicationData = this.reconfigurePublicationProps.reduce(
				getData,
				{}
			);

			$.ajax({
				url: this.submissionApiUrl,
				method: 'POST',
				context: this,
				headers: {
					'X-Csrf-Token': pkp.currentUser.csrfToken,
					'X-Http-Method-Override': 'PUT',
				},
				data: submissionData,
				error: this.ajaxErrorCallback,
				success(r) {
					if (!Object.keys(publicationData).length) {
						window.location.reload();
						return;
					}
					$.ajax({
						url: this.publicationApiUrl,
						method: 'POST',
						context: this,
						headers: {
							'X-Csrf-Token': pkp.currentUser.csrfToken,
							'X-Http-Method-Override': 'PUT',
						},
						data: publicationData,
						error: this.ajaxErrorCallback,
						success(r) {
							window.location.reload();
						},
					});
				},
			});
		},

		/**
		 * Save the submission for later
		 *
		 * Make sure all autosaves that are outstanding
		 * get saved first.
		 */
		saveForLater() {
			this.isLoading = true;

			this.addAutosaves();

			const waitForSaves = setInterval(() => {
				if (this.isAutosaving) {
					return;
				}
				if (this.isDisconnected) {
					clearInterval(waitForSaves);
					this.isLoading = false;
					this.openSaveForLaterFailed();
					return;
				}
				$.ajax({
					url: this.submissionApiUrl + '/saveForLater',
					method: 'POST',
					context: this,
					headers: {
						'X-Csrf-Token': pkp.currentUser.csrfToken,
						'X-Http-Method-Override': 'PUT',
					},
					data: {
						step: this.startedSteps[this.startedSteps.length - 1],
					},
					error() {
						this.isLoading = false;
						this.openSaveForLaterFailed();
					},
					success() {
						window.location = this.submissionSavedUrl;
					},
				});
				clearInterval(waitForSaves);
			}, 1000);
		},

		/**
		 * Update the message for when the last autosave occurred
		 */
		setLastAutosaveMessage() {
			if (!this.lastSavedTimestamp) {
				this.lastAutosavedMessage = '';
				return;
			}
			this.lastAutosavedMessage = this.i18nLastAutosaved.replace(
				'{$when}',
				moment(this.lastSavedTimestamp)
					.locale(this.getMomentLocale($.pkp.app.currentLocale))
					.fromNow()
			);
		},

		/**
		 * Update the contributors in the submission
		 */
		setContributors(newContributors) {
			this.publication.authors = newContributors;
		},

		/**
		 * Update the publication in the submission
		 */
		setPublication(newPublication) {
			this.publication = newPublication;
		},

		/**
		 * Complete the submission
		 */
		submit() {
			this.openDialog({
				name: 'submitConfirmation',
				title: this.i18nSubmit,
				message: this.i18nConfirmSubmit.replace(
					'{$title}',
					this.localize(this.publication.title)
				),
				actions: [
					{
						label: this.i18nSubmit,
						isPrimary: true,
						callback: () => {
							$.ajax({
								url: this.submitApiUrl,
								context: this,
								method: 'POST',
								headers: {
									'X-Csrf-Token': pkp.currentUser.csrfToken,
									'X-Http-Method-Override': 'PUT',
								},
								error(r) {
									if (!r.responseJSON) {
										this.ajaxErrorCallback();
									} else {
										this.errors = r.responseJSON;
									}
									this.$modal.hide('submitConfirmation');
								},
								success() {
									window.location = this.submissionWizardUrl;
								},
							});
						},
					},
					{
						label: this.__('common.cancel'),
						isWarnable: true,
						callback: () => this.$modal.hide('submitConfirmation'),
					},
				],
			});
		},

		/**
		 * Apply an autosave that was stored instead of saved
		 */
		restoreStoredAutosave(payload) {
			let form;
			const dataKeys = Object.keys(payload.data);
			this.steps.forEach((step) => {
				step.sections.forEach((section) => {
					if (section.type !== 'form' || section.form.id !== payload.id) {
						return;
					}
					form = section.form;
				});
			});
			if (!form) {
				return;
			}
			this.updateForm(payload.id, {
				fields: form.fields.map((field) => {
					if (!dataKeys.includes(field.name)) {
						return field;
					}
					return {
						...field,
						value: payload.data[field.name],
					};
				}),
			});
		},

		/**
		 * Update a form with new data
		 *
		 * This is fired every time a form field changes, so
		 * resource-intensive code should not be run every
		 * time this method is called.
		 */
		updateForm(formId, data) {
			this.steps.forEach((step, stepIndex) => {
				step.sections.forEach((section, sectionIndex) => {
					if (section.type !== 'form' || section.form.id !== formId) {
						return;
					}
					this.steps[stepIndex].sections[sectionIndex].form = {
						...this.steps[stepIndex].sections[sectionIndex].form,
						...data,
					};
					if (this.staleForms.indexOf(formId) === -1) {
						this.staleForms.push(formId);
					}
				});
			});
		},

		/**
		 * Validate the submission details
		 *
		 * Always wait for autosaves to complete before
		 * trying to validate
		 */
		validate() {
			this.isValidating = true;
			this.errors = {};

			const waitForSaves = setInterval(() => {
				if (this.isAutosaving || this.isDisconnected) {
					return;
				}
				clearInterval(waitForSaves);

				$.ajax({
					url: this.submitApiUrl,
					method: 'POST',
					context: this,
					headers: {
						'X-Csrf-Token': pkp.currentUser.csrfToken,
						'X-Http-Method-Override': 'PUT',
					},
					data: {
						_validateOnly: true,
					},
					error(r) {
						if (!r.responseJSON) {
							this.ajaxErrorCallback();
						} else {
							this.errors = r.responseJSON;
						}
					},
					success(r) {
						this.errors = {};
					},
					complete() {
						this.isValidating = false;
					},
				});
			}, 500);
		},
	},
	watch: {
		/**
		 * Update when the step changes
		 */
		currentStepIndex(newVal, oldVal) {
			if (newVal === oldVal) {
				return;
			}

			// Add forms to be autosaved
			this.addAutosaves();

			// Update the list of steps that have been started
			this.steps.forEach((step, i) => {
				if (
					!this.startedSteps.includes(step.id) &&
					i <= this.currentStepIndex
				) {
					this.startedSteps.push(step.id);
				}
			});

			// Track step changes in the title and browser history
			const step = this.steps[newVal];
			document.title = this.getPageTitle(step);
			if (step.id !== window.location.hash.replace('#', '')) {
				this.addHistory(step);
			}

			// Trigger validation on the review step
			if (newVal === this.steps.length - 1) {
				this.validate();
			}
		},

		/**
		 * Set form data when validation errors are changed
		 */
		errors(newVal, oldVal) {
			const keys = Object.keys(newVal);
			this.steps.forEach((step, stepIndex) => {
				step.sections.forEach((section, sectionIndex) => {
					if (section.type === 'form') {
						section.form.fields.forEach((field) => {
							if (keys.includes(field.name)) {
								this.steps[stepIndex].sections[sectionIndex].form.errors = {
									...this.steps[stepIndex].sections[sectionIndex].form.errors,
									...{[field.name]: newVal[field.name]},
								};
							}
						});
					}
				});
			});

			pkp.eventBus.$emit('submission:submit:errors', newVal, this);
		},

		/**
		 * Update the last autosave message as soon
		 * as autosave finishes
		 */
		isAutosaving(newVal, oldVal) {
			if (!newVal && oldVal) {
				this.setLastAutosaveMessage();
			}
		},
	},
	created() {
		if (!window.location.hash) {
			const newStep = this.steps.find(
				(step) => step.id === this.submission.submissionProgress
			);
			this.openStep(newStep ? newStep.id : this.steps[0].id);
		}

		/**
		 * Regularly update the last saved message
		 * so it shows an accurate time
		 */
		setInterval(this.setLastAutosaveMessage, 3000);
	},
};
</script>

<style lang="less">
@import '../../styles/_import';

.submissionWizard .app__pageHeading {
	display: flex;
	margin: 2rem 0 0.25rem;

	> .pkpButton {
		margin-inline-start: auto;
	}
}

.submissionWizard__steps {
	margin-top: 2rem;
}

.submissionWizard .pkpSteps__buttonWrapper {
	border: @bg-border-light;
	border-radius: @radius;
	margin-bottom: 2rem;
}

.submissionWizard__submissionDetails,
.submissionWizard__submissionConfiguration {
	font-size: @font-sml;
	line-height: @line-sml;
}

// Override the form locale switcher styles
.submissionWizard__stepForm .pkpFormLocales {
	border: none;
	margin-top: -1rem;
	margin-bottom: 1rem;
	padding-right: 0;

	.pkpFormLocales__locale--isPrimary {
		border: none;
	}
}

// Hide the form footer for each form, since
// buttons and errors are handled separately
.submissionWizard__stepForm .pkpFormPage__footer {
	display: none;
}

// Review step
.submissionWizard__reviewPanel {
	border: @bg-border-light;
	border-radius: @radius;

	+ .submissionWizard__reviewPanel {
		margin-top: 2rem;
	}
}

.submissionWizard__reviewPanel__header {
	display: flex;
	align-items: center;
	padding: 0.5rem 1rem;
	border-bottom: @bg-border-light;

	h3 {
		margin: 0;
		font-size: @font-base;
		line-height: @line-base;
	}
}

.submissionWizard__reviewPanel__edit {
	margin-inline-start: auto;
}

.submissionWizard__reviewPanel__body {
	font-size: @font-sml;
	line-height: @line-sml;
}

.submissionWizard__reviewPanel__list {
	margin: 0;
	padding: 0;
	list-style: none;

	li {
		display: flex;
		align-items: center;
		margin: 1rem;
	}

	.pkpBadge {
		white-space: nowrap;
	}
}

.submissionWizard__reviewPanel__list__name {
	overflow: hidden;
	text-overflow: ellipsis;
	white-space: nowrap;
}

.submissionWizard__reviewPanel__list__actions {
	margin-inline-start: auto;
	white-space: nowrap;

	> * {
		margin-inline-start: 0.5rem;
	}
}

.submissionWizard__review_errors {
	margin-bottom: 2rem;
}

.submissionWizard__reviewPanel__fileLink {
	display: block;
	margin-left: -0.25rem;
	padding: 0.25rem;
	border: 1px solid transparent;
	border-radius: @radius;
	color: @text;
	text-decoration: none;

	&:hover,
	&:focus {
		color: @text;
		border-color: @primary;
		outline: 0;
	}
}

.submissionWizard__reviewPanel__download {
	text-align: center;
}

.submissionWizard__reviewPanel__item {
	padding: 1rem;
	border-bottom: @bg-border-light;

	&:last-child {
		border-bottom: none;
	}

	.pkpNotification {
		margin-bottom: 0.5rem;
	}
}

.submissionWizard__reviewPanel__item__header,
.submissionWizard__reviewPanel__item__value {
	margin: 0;
	font-size: @font-sml;
	line-height: @line-sml;
}

.submissionWizard__reviewPanel__item__value {
	> p:first-child,
	> ul:first-child {
		margin-top: 0;
	}

	> p:last-child,
	> ul:last-child {
		margin-bottom: 0;
	}
}

.submissionWizard__reviewEmptyWarning {
	margin: 1rem;
}

// When the data being reviewed is a list of items
ul.submissionWizard__reviewPanel__item__value {
	margin: 0;
	padding: 0;
	list-style: none;
}

// A table used for a review
.submissionWizard__reviewPanel__body {
	.pkpTable {
		border: none;
	}
}

.submissionWizard__reviewPanel__citation
	+ .submissionWizard__reviewPanel__citation {
	margin-top: 1rem;
}

.submissionWizard__footer {
	padding: 2rem;
	background: @lift;
	border: @bg-border-light;
	border-top: none;
}

.submissionWizard__lastSaved {
	margin-right: 0.5rem;
	font-size: @font-sml;
	line-height: @line-sml;
}

.submissionWizard__lastSaved--disconnected {
	color: @no-red;

	.pkpSpinner:before {
		border-top-color: @no-red;
		border-left-color: @no-red;
	}
}

// Loading screen when review is opened
.submissionWizard__loadingReview {
	position: absolute;
	top: 0;
	left: 0;
	right: 0;
	bottom: 0;
	z-index: 2;
	transition: opacity 0.3s;
	background: rgba(255, 255, 255, 0.75);
	text-align: center;
	font-size: @font-sml;
}

.submissionWizard__reviewLoading-enter,
.submissionWizard__reviewLoading-leave-to {
	opacity: 0;
}
</style>