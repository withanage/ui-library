<template>
	<div>
		<FormFieldLabel :label="label" />
		<template v-for="group in assignableUserGroups" :key="group.id">
			<span class="flex gap-2">
				<input
					ref="input"
					v-model="userGroupIds"
					:value="group.id"
					class="col-span-1"
					name="userGroupIds"
					type="checkbox"
					@change="updateCurrentValue(group)"
				/>
				<label :for="group.id" class="col-span-1">{{ group.name }}</label>
			</span>
		</template>
	</div>
</template>

<script>
import FieldBase from '@/components/Form/fields/FieldBase.vue';
import FormFieldLabel from '@/components/Form/FormFieldLabel.vue';

export default {
	name: 'UserGroupIds',
	components: {FormFieldLabel},
	extends: FieldBase,
	props: {
		assignedUserGroupIds: {
			type: Array,
			required: true,
			default: () => [],
		},
		assignableUserGroups: {
			type: Array,
			required: true,
			default: () => [],
		},
	},
	data() {
		return {
			userGroupIds: [],
		};
	},
	computed: {},
	created() {},
	mounted() {
		this.userGroupIds = this.assignedUserGroupIds;
		this.currentValue = this.assignedUserGroupIds;
	},
	methods: {
		updateCurrentValue(group) {
			this.currentValue = this.userGroupIds;
		},
	},
};
</script>

<style lang="less">
@import '../../../styles/_import';
</style>
