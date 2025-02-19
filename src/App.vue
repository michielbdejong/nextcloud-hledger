<template>
	<Content app-name="hledger">
		<AppNavigation>
			<template #list>
				<AppNavigationItem title="Add Transactions" icon="icon-add" @click="startAddingTransactions" />
				<AppNavigationItem v-for="item in navigation.reports"
					:key="item.id"
					:title="item.title"
					:icon="item.icon"
					@click="getReport(item.name, item)" />
			</template>
			<template #footer>
				<AppNavigationSettings>
					<label for="hledger_folder">HLedger Folder</label>
					<input id="hledger_folder"
						v-model="settings.hledger_folder"
						type="text">
					<label for="journal_file">Journal File</label>
					<input id="journal_file"
						v-model="settings.journal_file"
						type="text">
					<label for="budget_file">Budget File</label>
					<input id="budget_file"
						v-model="settings.budget_file"
						type="text">
					<input type="submit" value="Save" @click="saveSettings">
				</AppNavigationSettings>
			</template>
		</AppNavigation>
		<AppContent>
			<table class="hledger-report">
				<tr v-for="row in report.data" :key="row.id">
					<td v-for="cell in row" :key="cell.id" :class="{ outline: shouldOutlineRow(row[0]), indent: shouldIndentCell(cell) }">
						<a v-if="isSubAccount(cell)" href="#" @click="getReport('accountregister', { account: cell })">{{ cell }}</a>
						<button v-else-if="cell === 'edit'" @click="editTransaction(row)">
							edit
						</button>
						<div v-else>
							{{ truncate(cell, 32) }}
						</div>
					</td>
				</tr>
			</table>
		</AppContent>
		<div>
			<Modal v-if="transaction.visible"
				class="hl-transaction"
				title="Add Transactions"
				@close="stopAddingTransactions">
				<div class="hledger-add-transactions">
					<div class="hlt-row">
						<DatetimePicker v-model="transaction.date" value-type="format" />
						<input v-model="transaction.code"
							type="text"
							class="t-code"
							placeholder="code (transaction)">
						<select v-model="transaction.status">
							<option value="" />
							<option value="!">
								!
							</option>
							<option value="*">
								*
							</option>
						</select>
					</div>
					<div class="hlt-row hlt-wrap">
						<input v-model="transaction.description"
							type="text"
							class="t-description"
							placeholder="description (transaction)">
						<input v-model="transaction.comment"
							type="text"
							class="t-comment"
							placeholder="comment (transaction)">
					</div>
					<ul class="postings">
						<li v-for="(posting, index) in transaction.postings" :key="posting.id">
							<div class="hlt-row hlt-wrap">
								<div class="hlt-row p-left">
									<VueAutosuggest v-model="posting.account"
										:suggestions="filterAccounts(posting.account)"
										:input-props="{id:'p'+index+'__input', class:'p-account', placeholder:'account'}"
										@selected="(suggestion) => accountSelected(index, suggestion.item)">
										<template slot-scope="{suggestion}">
											<span>{{ suggestion.item }}</span>
										</template>
									</VueAutosuggest>
									<input v-model="posting.amount"
										type="text"
										class="p-amount"
										placeholder="amount">
								</div>
								<div class="hlt-row p-right">
									<select v-model="posting.status">
										<option value="" />
										<option value="!">
											!
										</option>
										<option value="*">
											*
										</option>
									</select>
									<input v-model="posting.comment"
										type="text"
										class="p-comment"
										placeholder="comment (posting)">
									<button v-if="index > 1" @click="removePosting(index)">
										X
									</button>
								</div>
							</div>
						</li>
					</ul>
					<button @click="addPosting">
						Add Posting
					</button>
					<button :disabled="cantBalanceTransaction" @click="balanceTransaction">
						Balance
					</button>
					<button :disabled="transactionInvalid" @click="addTransaction">
						Save Transaction
					</button>
					<div class="t-validation">
						{{ transactionInvalid }}
					</div>
				</div>
			</Modal>
		</div>
	</Content>
</template>
<script>
import Content from '@nextcloud/vue/dist/Components/Content'
import AppNavigation from '@nextcloud/vue/dist/Components/AppNavigation'
import AppNavigationItem from '@nextcloud/vue/dist/Components/AppNavigationItem'
import AppNavigationSettings from '@nextcloud/vue/dist/Components/AppNavigationSettings'
import AppContent from '@nextcloud/vue/dist/Components/AppContent'
import Modal from '@nextcloud/vue/dist/Components/Modal'
import DatetimePicker from '@nextcloud/vue/dist/Components/DatetimePicker'
import '@nextcloud/dialogs/styles/toast.scss'
import { generateUrl } from '@nextcloud/router'
import { showError, showSuccess } from '@nextcloud/dialogs'
import axios from '@nextcloud/axios'
import { VueAutosuggest } from 'vue-autosuggest'
export default {
	name: 'App',
	components: {
		Content,
		AppNavigation,
		AppNavigationItem,
		AppNavigationSettings,
		AppContent,
		Modal,
		DatetimePicker,
		VueAutosuggest,
	},
	data() {
		const state = OCP.InitialState.loadState('hledger', 'state')
		this.initializeTransaction(state)
		return state
	},
	computed: {
		transactionInvalid() {
			if (!this.transaction.code && !this.transaction.description) {
				return 'Transaction code or description required'
			}
			let sum = 0
			const units = []
			for (let i = 0; i < this.transaction.postings.length; i++) {
				const posting = this.transaction.postings[i]
				if (!posting.account) {
					return 'Posting ' + (i + 1) + ' account required'
				}
				const amount = this.parseAmount(posting.amount)
				if (!amount) {
					return 'Posting ' + (i + 1) + ' numeric amount required'
				}
				sum += amount[0]
				const unit = (amount.length > 1) ? amount[1] : ''
				if (!units.includes(unit)) {
					units.push(unit)
				}
			}
			if (units.length < 2 && Math.abs(sum) > 0.001) {
				return 'Transaction does not balance: ' + sum.toFixed(2)
			}
			return null
		},
		cantBalanceTransaction() {
			let amountsEntered = 0
			let amountsBlank = 0
			const units = []
			for (let i = 0; i < this.transaction.postings.length; i++) {
				const posting = this.transaction.postings[i]
				const amount = this.parseAmount(posting.amount)
				if (posting.amount.trim() === '') {
					amountsBlank++
				} else if (amount) {
					amountsEntered++
					const unit = amount.length > 1 ? amount[1] : ''
					if (!units.includes(unit)) {
						units.push(unit)
					}
				}
			}
			return (units.length > 1) || (amountsBlank !== 1) || (amountsEntered !== this.transaction.postings.length - 1)
		},
	},
	methods: {
		apiUrl(x) {
			return generateUrl('/apps/hledger/api/1/' + x)
		},
		truncate(text, stop, clamp) {
			return text.slice(0, stop) + (stop < text.length ? clamp || '...' : '')
		},
		isSubAccount(x) {
			return x.match(/^(assets|liabilities|equity|income|expenses):/g)
		},
		shouldOutlineRow(x) {
			return ['Account', 'Total:'].includes(x)
		},
		shouldIndentCell(x) {
			return ['Account', '<unbudgeted>'].includes(x) || this.isSubAccount(x)
		},
		startAddingTransactions() {
			this.initializeTransaction(this)
			this.transaction.visible = true
		},
		stopAddingTransactions() {
			this.transaction.visible = false
			this.reloadReport()
		},
		addPosting() {
			this.transaction.postings.push(this.newPosting())
		},
		removePosting(index) {
			this.transaction.postings.splice(index, 1)
		},
		parseAmount(entered) {
			const split = entered.trim().split(/\s+/)
			if (!entered || isNaN(split[0])) {
				return null
			}
			const parsed = [parseFloat(split[0])]
			if (split.length > 1) {
				parsed.push(split[1])
			}
			return parsed
		},
		balanceTransaction() {
			let sum = 0
			let emptyAmountIndex = null
			for (let i = 0; i < this.transaction.postings.length; i++) {
				const posting = this.transaction.postings[i]
				const amount = this.parseAmount(posting.amount)
				if (posting.amount.trim() === '') {
					emptyAmountIndex = i
				} else if (amount) {
					sum += amount[0]
				}
			}
			if (emptyAmountIndex !== null) {
				this.transaction.postings[emptyAmountIndex].amount = (-sum).toFixed(2)
			}
		},
		async addTransaction() {
			try {
				await axios.post(this.apiUrl('transaction'), this.transaction)
				const snippet = this.truncate((this.transaction.code + ' ' + this.transaction.description).trim(), 20)
				showSuccess(t('hledger', 'Saved ' + snippet))
				this.updateNewAccounts(this)
				this.initializeTransaction(this)
			} catch (e) {
				showError(t('hledger', 'Error saving transaction: ' + e.message))
			}
		},
		updateNewAccounts(state) {
			for (let i = 0; i < state.transaction.postings.length; i++) {
				const posting = state.transaction.postings[i]
				if (!state.accounts.includes(posting.account)) {
					state.accounts.push(posting.account)
				}
			}
		},
		initializeTransaction(state) {
			if (!('transaction' in state)) {
				state.transaction = {
					visible: false,
					date: new Date().toISOString().split('T')[0],
					postings: [],
				}
			}
			state.transaction.status = ''
			state.transaction.code = ''
			state.transaction.description = ''
			state.transaction.comment = ''
			state.transaction.postings.splice(0, state.transaction.postings.length)
			state.transaction.postings.push(this.newPosting())
			state.transaction.postings.push(this.newPosting())
		},
		newPosting() {
			return {
				status: '',
				account: '',
				amount: '',
				comment: '',
			}
		},
		filterAccounts(input) {
			return [{
				data: this.accounts.filter(account => {
					return input && account.toLowerCase().includes(input.toLowerCase())
				}),
			}]
		},
		accountSelected(posting, account) {
			this.transaction.postings[posting].account = account
		},
		reloadReport() {
			if (this.report.name) {
				this.getReport(this.report.name, this.report.options)
			} else {
				this.getReport('balancesheet')
			}
		},
		async getReport(report, options) {
			try {
				const getOptions = { params: { } }
				if (options.customName !== undefined) {
					getOptions.params.reportName = options.customName
				}
				if (options.includesFiles !== undefined && !options.includesFiles) {
					getOptions.params.includeDefaultFiles = true
				}
				if (options.account !== undefined) {
					getOptions.params.account = options.account
				}
				this.report.data = (await axios.get(this.apiUrl(report), getOptions)).data
				this.report.name = report
				this.report.options = options

				/* For register-style reports, add "edit" button and scroll to bottom */
				if (this.report.data.length > 0 && this.report.data[0][0].trim() === 'date' && this.report.data[0][2].trim() === 'description') {
					this.report.data = this.report.data.map(function(row) {
						return row.concat([row[0].trim() === 'date' ? '' : 'edit'])
					})
					const self = this
					setTimeout(function() { self.scrollToBottom('html') }, 37)
				}
			} catch (e) {
				showError(t('hledger', 'Error getting ' + report + ': ' + e.message))
			}
		},
		scrollToBottom(selector) {
			const main = window.jQuery(selector)
			main.scrollTop(main.prop('scrollHeight'))
		},
		editTransaction(row) {
			const self = this
			const journalPath = '/' + this.settings.hledger_folder + '/' + this.settings.journal_file
			OCA.Viewer.open({
				path: journalPath,
				async onClose() {
					showSuccess(t('hledger', 'HLedger journal saved'))
					self.reloadReport()
				},
			})

			let attempts = 0
			const interval = setInterval(function() {
				const found = self.findEditorElements()
				if (!found) {
					attempts++
					if (attempts === 30) {
						clearInterval(interval)
						showError(t('hledger', 'Failed to open ' + journalPath))
					}
					return
				}

				clearInterval(interval)
				self.goToTransaction(found.editor, found.code, row[0], row[2])
			}, 1000)
		},
		findEditorElements() {
			const jqeditor = window.jQuery('#editor')
			if (jqeditor.length === 0) {
				return null
			}
			const jqcode = window.jQuery('code', jqeditor)
			if (jqcode.length === 0) {
				return null
			}
			return {
				editor: jqeditor.get(0),
				code: jqcode.get(0),
			}
		},
		goToTransaction(editor, code, date, description) {
			const textnode = this.findTextNodeInElement(code)
			if (!textnode) {
				showError(t('hledger', 'Selecting transaction, can\'t find text node in editor.'))
				return
			}

			const pattern = '^.*' + this.regexEscape(date) + '.+' + this.regexEscape(description) + '.*$'
			const match = (new RegExp(pattern, 'gm')).exec(textnode.wholeText)
			if (!match) {
				showError(t('hledger', 'No transaction with ' + date + ' and ' + description))
				return
			}

			const range = document.createRange()
			range.setStart(textnode, match.index)
			range.setEnd(textnode, match.index + match[0].length)

			const selection = window.getSelection()
			selection.removeAllRanges()
			selection.addRange(range)

			editor.scroll(0, range.getBoundingClientRect().top - code.getBoundingClientRect().top)
		},
		findTextNodeInElement(element) {
			for (let i = 0; i < element.childNodes.length; i++) {
				const child = element.childNodes[i]
				if (child.nodeType === Node.TEXT_NODE) {
					return child
				} else if (child.nodeType === Node.ELEMENT_NODE) {
					const found = this.findTextNodeInElement(child)
					if (found) {
						return found
					}
				}
			}
			return null
		},
		regexEscape(x) {
			return x.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
		},
		async saveSettings() {
			try {
				await axios.put(this.apiUrl('settings'), this.settings)
				showSuccess(t('hledger', 'Settings saved'))
			} catch (e) {
				showError(t('hledger', 'Error saving settings'))
			}
		},
	},
}
</script>
