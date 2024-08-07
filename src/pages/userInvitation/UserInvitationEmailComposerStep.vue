<template>
	<Composer
		:id="store.currentStep.id"
		:add-c-c-label="t('common.addCCBCC')"
		:attach-files-label="t('common.attachFiles')"
		:attached-files-label="t('common.attachedFiles')"
		:attachers="props.email.attachers"
		:attachments="props.email.attachments"
		:bcc="props.email.bcc"
		:bcc-label="t('email.bcc')"
		:body-label="t('stageParticipants.notify.message')"
		:can-change-recipients="props.email.canChangeRecipients"
		:cc="props.email.cc"
		:cc-label="t('email.cc')"
		:confirm-switch-locale-label="t('email.confirmSwitchLocale')"
		:deselect-label="t('common.deselect')"
		:email-templates="props.email.emailTemplates"
		:email-templates-api-url="store.emailTemplatesApiUrl"
		:errors="sectionErrors.emailComposer"
		:find-template-label="t('common.findTemplate')"
		:initial-template-key="props.email.initialTemplateKey"
		:insert-label="t('common.insert')"
		:insert-modal-label="t('common.insertContent')"
		:insert-content-label="t('common.content')"
		:insert-search-label="t('common.insertContentSearch')"
		:load-template-label="t('common.emailTemplates')"
		:locale="props.email.locale"
		:locales="props.email.locales"
		:more-search-results-label="t('common.numberedMore')"
		:recipient-options="props.email.recipientOptions"
		:recipients="props.email.recipients"
		:recipients-label="t('email.to')"
		:remove-item-label="t('common.removeItem')"
		:searching-label="t('common.searching')"
		:search-results-label="t('search.searchResults')"
		:subject-label="t('email.subject')"
		:switch-to-label="t('common.switchTo')"
		:switch-to-named-language-label="t('common.switchToNamedItem')"
		:variables="props.email.variables"
		v-bind="emailComposer"
		@set="(componentName, update) => updateEmail(update)"
	></Composer>
</template>

<script setup>
import {computed} from 'vue';
import Composer from '@/components/Composer/Composer.vue';
import {useTranslation} from '@/composables/useTranslation';
import {defineProps} from 'vue';
import {useUserInvitationPageStore} from './UserInvitationPageStore';

const props = defineProps({
	email: {type: Object, required: true},
	validateFields: {
		type: Array,
		default() {
			return null;
		},
	},
});
const {t} = useTranslation();

const store = useUserInvitationPageStore();
const emailComposer = computed(() => store.invitationPayload.emailComposer);

function updateEmail(update) {
	const emailComposerUpdate = {
		...store.invitationPayload.emailComposer,
		...update,
	};
	store.updatePayload('emailComposer', emailComposerUpdate);
}

if (!store.invitationPayload.body) {
	updateEmail({
		subject: props.email.subject,
		body: props.email.body,
	});
}

const sectionErrors = computed(() => {
	return props.validateFields.reduce((obj, key) => {
		if (store.errors[key]) {
			obj[key] = store.errors[key];
		}
		return obj;
	}, {});
});
</script>
