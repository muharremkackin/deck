<!--
  - @copyright Copyright (c) 2018 Julius Härtl <jus@bitgrid.net>
  -
  - @author Julius Härtl <jus@bitgrid.net>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<AppSidebar v-if="currentBoard && currentCard && copiedCard"
		:title="currentCard.title"
		:subtitle="subtitle"
		@close="closeSidebar">
		<template #secondary-actions />
		<AppSidebarTab id="details"
			:order="0"
			:name="t('deck', 'Details')"
			icon="icon-home">
			<div class="section-wrapper">
				<div v-tooltip="t('deck', 'Tags')" class="section-label icon-tag">
					<span class="hidden-visually">{{ t('deck', 'Tags') }}</span>
				</div>
				<div class="section-details">
					<Multiselect v-model="allLabels"
						:multiple="true"
						:disabled="!canEdit"
						:options="currentBoard.labels"
						:placeholder="t('deck', 'Assign a tag to this card…')"
						:taggable="true"
						label="title"
						track-by="id"
						@select="addLabelToCard"
						@remove="removeLabelFromCard">
						<template #option="scope">
							<div :style="{ backgroundColor: '#' + scope.option.color, color: textColor(scope.option.color)}" class="tag">
								{{ scope.option.title }}
							</div>
						</template>
						<template #tag="scope">
							<div :style="{ backgroundColor: '#' + scope.option.color, color: textColor(scope.option.color)}" class="tag">
								{{ scope.option.title }}
							</div>
						</template>
					</Multiselect>
				</div>
			</div>

			<div class="section-wrapper">
				<div v-tooltip="t('deck', 'Assign to users')" class="section-label icon-group">
					<span class="hidden-visually">{{ t('deck', 'Assign to users') }}</span>
				</div>
				<div class="section-details">
					<Multiselect v-model="assignedUsers"
						:disabled="!canEdit"
						:multiple="true"
						:options="assignableUsers"
						:placeholder="t('deck', 'Assign a user to this card…')"
						label="displayname"
						track-by="primaryKey"
						@select="assignUserToCard"
						@remove="removeUserFromCard">
						<template #option="scope">
							<Avatar :user="scope.option.primaryKey" />
							<span class="avatarLabel">{{ scope.option.displayname }} </span>
						</template>
					</Multiselect>
				</div>
			</div>

			<div class="section-wrapper">
				<div v-tooltip="t('deck', 'Due date')" class="section-label icon-calendar-dark">
					<span class="hidden-visually">{{ t('deck', 'Due date') }}</span>
				</div>
				<div class="section-details">
					<DatetimePicker v-model="copiedCard.duedate"
						:placeholder="t('deck', 'Set a due date')"
						type="datetime"
						lang="en"
						:disabled="!canEdit"
						format="YYYY-MM-DD HH:mm"
						confirm
						@change="setDue()" />
					<Actions v-if="canEdit">
						<ActionButton v-if="copiedCard.duedate" icon="icon-delete" @click="removeDue()">
							{{ t('deck', 'Remove due date') }}
						</ActionButton>
					</Actions>
				</div>
			</div>

			<div class="section-wrapper">
				<CollectionList v-if="currentCard.id"
					:id="`${currentCard.id}`"
					:name="currentCard.title"
					type="deck-card" />
			</div>

			<h5>
				{{ t('deck', 'Description') }}
				<a v-tooltip="t('deck', 'Formatting help')"
					href="https://deck.readthedocs.io/en/latest/Markdown/"
					target="_blank"
					class="icon icon-info" />
			</h5>
			<!-- FIXME: make sure the editor is disabled when canEdit is false -->
			<VueEasymde ref="markdownEditor" v-model="copiedCard.description" :configs="mdeConfig" />
		</AppSidebarTab>

		<AppSidebarTab id="attachments"
			:order="1"
			:name="t('deck', 'Attachments')"
			icon="icon-attach">
			<CardSidebarTabAttachments :card="currentCard" />
		</AppSidebarTab>

		<AppSidebarTab v-if="hasComments"
			id="comments"
			:order="2"
			:name="t('deck', 'Comments')"
			icon="icon-comment">
			<CardSidebarTabComments :card="currentCard" />
		</AppSidebarTab>

		<AppSidebarTab v-if="hasActivity"
			id="timeline"
			:order="3"
			:name="t('deck', 'Timeline')"
			icon="icon-activity">
			<CardSidebarTabActivity :card="currentCard" />
		</AppSidebarTab>
	</AppSidebar>
</template>

<script>
import { Avatar, Actions, ActionButton, Multiselect, AppSidebar, AppSidebarTab, DatetimePicker } from '@nextcloud/vue'
import { mapState, mapGetters } from 'vuex'
import VueEasymde from 'vue-easymde/dist/VueEasyMDE.common'
import Color from '../../mixins/color'
import { CollectionList } from 'nextcloud-vue-collections'
import CardSidebarTabAttachments from './CardSidebarTabAttachments'
import CardSidebarTabComments from './CardSidebarTabComments'
import CardSidebarTabActivity from './CardSidebarTabActivity'

const capabilities = window.OC.getCapabilities()

export default {
	name: 'CardSidebar',
	components: {
		AppSidebar,
		AppSidebarTab,
		Multiselect,
		DatetimePicker,
		VueEasymde,
		Actions,
		ActionButton,
		Avatar,
		CollectionList,
		CardSidebarTabAttachments,
		CardSidebarTabComments,
		CardSidebarTabActivity,
	},
	mixins: [
		Color,
	],
	props: {
		id: {
			type: Number,
			required: true,
		},
	},
	data() {
		return {
			assignedUsers: null,
			addedLabelToCard: null,
			copiedCard: null,
			allLabels: null,
			desc: null,
			mdeConfig: {
				autoDownloadFontAwesome: false,
				spellChecker: false,
				autofocus: true,
				autosave: { enabled: true, uniqueId: 'unique' },
				toolbar: false,
			},
			lastModifiedRelative: null,
			lastCreatedRemative: null,
			hasActivity: capabilities && capabilities.activity,
			hasComments: window.OCP && window.OCP.Comments,
		}
	},
	computed: {
		...mapState({
			currentBoard: state => state.currentBoard,
			assignableUsers: state => state.assignableUsers,
		}),
		...mapGetters(['canEdit']),
		currentCard() {
			return this.$store.getters.cardById(this.id)
		},
		subtitle() {
			return t('deck', 'Modified') + ': ' + this.lastModifiedRelative + ' ' + t('deck', 'Created') + ': ' + this.lastCreatedRemative
		},
	},
	watch: {
		'currentCard': {
			immediate: true,
			handler() {
				if (!this.currentCard) {
					return
				}
				this.copiedCard = JSON.parse(JSON.stringify(this.currentCard))
				this.allLabels = this.currentCard.labels

				if (this.currentCard.assignedUsers.length > 0) {
					this.assignedUsers = this.currentCard.assignedUsers.map((item) => item.participant)
				} else {
					this.assignedUsers = []
				}

				this.desc = this.currentCard.description
				this.updateRelativeTimestamps()
			},
		},

		'copiedCard.description': function() {
			this.saveDesc()
		},
	},
	created() {
		setInterval(this.updateRelativeTimestamps, 10000)
	},
	destroyed() {
		clearInterval(this.updateRelativeTimestamps)
	},
	methods: {
		updateRelativeTimestamps() {
			this.lastModifiedRelative = OC.Util.relativeModifiedDate(this.currentCard.lastModified * 1000)
			this.lastCreatedRemative = OC.Util.relativeModifiedDate(this.currentCard.createdAt * 1000)
		},
		setDue() {
			this.$store.dispatch('updateCardDue', this.copiedCard)
		},
		removeDue() {
			this.copiedCard.duedate = null
			this.$store.dispatch('updateCardDue', this.copiedCard)
		},
		saveDesc() {
			this.$store.dispatch('updateCardDesc', this.copiedCard)
		},

		closeSidebar() {
			this.$router.push({ name: 'board' })
		},

		assignUserToCard(user) {
			this.copiedCard.newUserUid = user.uid
			this.$store.dispatch('assignCardToUser', this.copiedCard)
		},

		removeUserFromCard(user) {
			this.copiedCard.removeUserUid = user.uid
			this.$store.dispatch('removeUserFromCard', this.copiedCard)
		},

		addLabelToCard(newLabel) {
			this.copiedCard.labels.push(newLabel)
			const data = {
				card: this.copiedCard,
				labelId: newLabel.id,
			}
			this.$store.dispatch('addLabel', data)
		},

		removeLabelFromCard(removedLabel) {

			const removeIndex = this.copiedCard.labels.findIndex((label) => {
				return label.id === removedLabel.id
			})
			if (removeIndex !== -1) {
				this.copiedCard.labels.splice(removeIndex, 1)
			}

			const data = {
				card: this.copiedCard,
				labelId: removedLabel.id,
			}
			this.$store.dispatch('removeLabel', data)
		},
	},
}
</script>

<style>
	@import "~easymde/dist/easymde.min.css";
	.vue-easymde, .CodeMirror {
		border: none;
	}
	.editor-preview,
	.editor-statusbar {
		display: none;
	}

	#app-sidebar .app-sidebar-header__desc h4 {
		font-size: 12px !important;
	}
</style>

<style lang="scss" scoped>

	h5 {
		border-bottom: 1px solid var(--color-border);
		margin-top: 20px;
		margin-bottom: 5px;
		color: var(--color-text-maxcontrast);

		.icon-info {
			display: inline-block;
			width: 16px;
			height: 16px;
			float: right;
			opacity: .7;
		}
	}

	aside::v-deep section {
		display: flex;
		flex-direction: column;
	}

	.section-wrapper {
		display: flex;
		max-width: 100%;
		margin-top: 10px;

		.section-label {
			background-position: 0px center;
			width: 28px;
			margin-left: 9px;
			flex-shrink: 0;
		}

		.section-details {
			flex-grow: 1;

			button.action-item--single {
				margin-top: -6px;
			}
		}
	}

	.tag {
		flex-grow: 0;
		flex-shrink: 1;
		overflow: hidden;
		padding: 1px 3px;
		border-radius: 3px;
		font-size: 85%;
		margin-right: 3px;
	}

	.avatarLabel {
		padding: 6px
	}

</style>
