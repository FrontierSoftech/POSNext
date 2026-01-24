<template>
	<Dialog v-model="show" :options="{ title: __('New Customer'), size: 'lg' }">
		<template #body-content>
			<div class="flex flex-col gap-4 max-h-[70vh] overflow-y-auto px-1">
				<!-- GSTIN / UIN -->
				<div>
					<label class="block text-start text-sm font-medium text-gray-700 mb-1">
						{{ __("GSTIN / UIN") }}
					</label>
					<div class="flex gap-2">
						<input
							v-model="customerData.gstin"
							type="text"
							:placeholder="__('Enter 15-digit GSTIN')"
							maxlength="15"
							class="flex-1 px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 text-start uppercase"
							@input="customerData.gstin = customerData.gstin.toUpperCase()"
						/>
						<button
							type="button"
							@click="fetchGSTINInfo"
							:disabled="!customerData.gstin || customerData.gstin.length !== 15 || fetchingGSTIN"
							class="px-4 py-2 bg-blue-500 text-white rounded-lg text-sm font-medium hover:bg-blue-600 disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors"
						>
							{{ fetchingGSTIN ? __("...") : __("Fetch") }}
						</button>
					</div>
					<p v-if="gstinStatus" class="mt-1 text-xs" :class="gstinStatus.includes('Status') || gstinStatus.includes('success') ? 'text-green-600' : 'text-red-600'">
						{{ gstinStatus }}
					</p>
				</div>

				<!-- Customer Name -->
				<div>
					<label class="block text-start text-sm font-medium text-gray-700 mb-1">
						{{ __("Customer Name") }} <span class="text-red-500">*</span>
					</label>
					<input
						v-model="customerData.customer_name"
						type="text"
						:placeholder="__('customer_name')"
						required
						class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
					/>
				</div>

				<!-- Customer Type -->
				<div>
					<label class="block text-start text-sm font-medium text-gray-700 mb-1">
						{{ __("Customer Type") }}
					</label>
					<select
						v-model="customerData.customer_type"
						class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
					>
						<option v-for="type in customerTypes" :key="type" :value="type">
							{{ type }}
						</option>
					</select>
				</div>

				<!-- Profession (Custom Field) -->
				<div>
					<label class="block text-start text-sm font-medium text-gray-700 mb-1">
						{{ __("Profession") }} <span class="text-red-500">*</span>
					</label>
					<select
						v-model="customerData.custom_profession"
						class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
					>
						<option value="">{{ __("Select Profession") }}</option>
						<option v-for="prof in professions" :key="prof" :value="prof">
							{{ prof }}
						</option>
					</select>
				</div>

				<!-- GST Category -->
				<div>
					<label class="block text-start text-sm font-medium text-gray-700 mb-1">
						{{ __("GST Category") }}
					</label>
					<select
						v-model="customerData.gst_category"
						class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
					>
						<option v-for="category in gstCategories" :key="category" :value="category">
							{{ category }}
						</option>
					</select>
				</div>

				<!-- Email ID -->
				<div>
					<label class="block text-start text-sm font-medium text-gray-700 mb-1">
						{{ __("Email ID") }}
					</label>
					<input
						v-model="customerData.email_id"
						type="email"
						:placeholder="__('_email_id')"
						class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
					/>
				</div>

				<!-- Mobile Number -->
				<div>
					<label class="block text-start text-sm font-medium text-gray-700 mb-1">
						{{ __("Mobile Number") }}
					</label>
					<div class="flex gap-2">
						<!-- Country Code Dropdown -->
						<div class="relative" ref="dropdownRef">
							<button
								type="button"
								@click="showCountryDropdown = !showCountryDropdown"
								class="flex items-center gap-1 w-24 ps-2 pe-1 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white hover:bg-gray-50"
							>
								<img
									:src="`https://flagcdn.com/h24/${currentCountryCode}.png`"
									:alt="currentCountryCode"
									class="w-6 h-auto rounded-sm"
									@error="handleFlagError"
								/>
								<span class="flex-1 text-start">{{ selectedCountryCode || "+91" }}</span>
								<svg class="w-4 h-4 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
								</svg>
							</button>

							<!-- Country Search Dropdown -->
							<div
								v-if="showCountryDropdown"
								class="absolute start-0 z-50 mt-1 w-80 max-h-80 bg-white rounded-lg shadow-lg border border-gray-200 overflow-hidden"
							>
								<div class="sticky top-0 bg-white border-b border-gray-200 p-2">
									<input
										ref="countrySearchRef"
										v-model="countrySearchQuery"
										type="text"
										:placeholder="__('Search country...')"
										class="w-full px-3 py-2 text-sm border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
										@keydown.escape="showCountryDropdown = false"
									/>
								</div>
								<div class="overflow-y-auto max-h-64">
									<button
										v-for="country in filteredCountries"
										:key="country.code"
										type="button"
										@click="selectCountry(country)"
										class="w-full flex items-center gap-3 px-3 py-2.5 hover:bg-gray-50 transition-colors text-start"
										:class="{ 'bg-blue-50': selectedCountryCode === country.isd }"
									>
										<img
											:src="`https://flagcdn.com/h24/${country.code.toLowerCase()}.png`"
											:alt="country.name"
											class="w-6 h-auto rounded-sm shadow-sm"
											@error="(e) => (e.target.style.display = 'none')"
										/>
										<span class="flex-1 text-sm font-medium text-gray-700">{{ country.name }}</span>
										<span class="text-sm text-gray-500">{{ country.isd }}</span>
									</button>
									<div v-if="filteredCountries.length === 0" class="px-4 py-8 text-center text-sm text-gray-500">
										{{ __("No countries found") }}
									</div>
								</div>
							</div>
						</div>

						<!-- Phone Number Input -->
						<input
							v-model="phoneNumber"
							type="tel"
							:placeholder="__('_mobile_no')"
							class="flex-1 px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 text-start"
							@input="updateMobileNumber"
						/>
					</div>
				</div>

				<!-- Primary Address Details Section -->
				<div class="border-t border-gray-200 pt-4 mt-2">
					<h3 class="text-sm font-semibold text-gray-800 mb-3">{{ __("Primary Address Details") }}</h3>

					<!-- Postal Code with search -->
					<div class="mb-3">
						<label class="block text-start text-sm font-medium text-gray-700 mb-1">
							{{ __("Postal Code") }}
						</label>
						<div class="relative">
							<input
								v-model="addressData.pincode"
								type="text"
								:placeholder="__('Begin typing for results.')"
								class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
								@input="onPincodeInput"
							/>
							<!-- Pincode suggestions dropdown -->
							<div
								v-if="pincodeSuggestions.length > 0 && showPincodeSuggestions"
								class="absolute z-50 mt-1 w-full bg-white rounded-lg shadow-lg border border-gray-200 max-h-48 overflow-y-auto"
							>
								<button
									v-for="suggestion in pincodeSuggestions"
									:key="suggestion.pincode"
									type="button"
									@click="selectPincode(suggestion)"
									class="w-full px-3 py-2 text-start text-sm hover:bg-gray-50"
								>
									{{ suggestion.pincode }} - {{ suggestion.city }}, {{ suggestion.state }}
								</button>
							</div>
						</div>
					</div>

					<!-- Address Line 1 -->
					<div class="mb-3">
						<label class="block text-start text-sm font-medium text-gray-700 mb-1">
							{{ __("Address Line 1") }} <span class="text-red-500">*</span>
						</label>
						<input
							v-model="addressData.address_line1"
							type="text"
							:placeholder="__('address_line1')"
							class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
						/>
					</div>

					<!-- Address Line 2 -->
					<div class="mb-3">
						<label class="block text-start text-sm font-medium text-gray-700 mb-1">
							{{ __("Address Line 2") }}
						</label>
						<input
							v-model="addressData.address_line2"
							type="text"
							:placeholder="__('address_line2')"
							class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
						/>
					</div>

					<!-- City/Town -->
					<div class="mb-3">
						<label class="block text-start text-sm font-medium text-gray-700 mb-1">
							{{ __("City/Town") }} <span class="text-red-500">*</span>
						</label>
						<input
							v-model="addressData.city"
							type="text"
							:placeholder="__('city')"
							class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
						/>
					</div>

					<!-- State/Province with search -->
					<div class="mb-3">
						<label class="block text-start text-sm font-medium text-gray-700 mb-1">
							{{ __("State/Province") }}
						</label>
						<div class="relative" ref="stateDropdownRef">
							<input
								v-model="stateSearchQuery"
								type="text"
								:placeholder="__('Begin typing for results.')"
								class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
								@focus="showStateDropdown = true"
								@input="showStateDropdown = true"
							/>
							<div
								v-if="showStateDropdown && filteredStates.length > 0"
								class="absolute z-50 mt-1 w-full bg-white rounded-lg shadow-lg border border-gray-200 max-h-48 overflow-y-auto"
							>
								<button
									v-for="state in filteredStates"
									:key="state"
									type="button"
									@click="selectState(state)"
									class="w-full px-3 py-2 text-start text-sm hover:bg-gray-50"
									:class="{ 'bg-blue-50': addressData.state === state }"
								>
									{{ state }}
								</button>
							</div>
						</div>
					</div>

					<!-- Country with search -->
					<div class="mb-3">
						<label class="block text-start text-sm font-medium text-gray-700 mb-1">
							{{ __("Country") }} <span class="text-red-500">*</span>
						</label>
						<div class="relative" ref="countryAddressDropdownRef">
							<input
								v-model="countryAddressSearchQuery"
								type="text"
								:placeholder="__('Begin typing for results.')"
								class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
								@focus="showCountryAddressDropdown = true"
								@input="showCountryAddressDropdown = true"
							/>
							<div
								v-if="showCountryAddressDropdown && filteredAddressCountries.length > 0"
								class="absolute z-50 mt-1 w-full bg-white rounded-lg shadow-lg border border-gray-200 max-h-48 overflow-y-auto"
							>
								<button
									v-for="country in filteredAddressCountries"
									:key="country.name"
									type="button"
									@click="selectAddressCountry(country)"
									class="w-full px-3 py-2 text-start text-sm hover:bg-gray-50 flex items-center gap-2"
									:class="{ 'bg-blue-50': addressData.country === country.name }"
								>
									<img
										:src="`https://flagcdn.com/h24/${country.code.toLowerCase()}.png`"
										:alt="country.name"
										class="w-5 h-auto rounded-sm"
										@error="(e) => (e.target.style.display = 'none')"
									/>
									{{ country.name }}
								</button>
							</div>
						</div>
					</div>
				</div>
			</div>
		</template>

		<template #actions>
			<div class="flex flex-col gap-2">
				<!-- Permission Warning -->
				<div v-if="!hasPermission" class="px-3 py-2 bg-amber-50 border border-amber-200 rounded-lg">
					<div class="flex items-start gap-2">
						<svg class="w-5 h-5 text-amber-600 flex-shrink-0 mt-0.5" fill="currentColor" viewBox="0 0 20 20">
							<path
								fill-rule="evenodd"
								d="M8.257 3.099c.765-1.36 2.722-1.36 3.486 0l5.58 9.92c.75 1.334-.213 2.98-1.742 2.98H4.42c-1.53 0-2.493-1.646-1.743-2.98l5.58-9.92zM11 13a1 1 0 11-2 0 1 1 0 012 0zm-1-8a1 1 0 00-1 1v3a1 1 0 002 0V6a1 1 0 00-1-1z"
								clip-rule="evenodd"
							/>
						</svg>
						<div class="flex-1">
							<p class="text-sm font-medium text-amber-900">{{ __("Permission Required") }}</p>
							<p class="text-xs text-amber-700 mt-0.5">
								{{ __("You don't have permission to create customers. Contact your administrator.") }}
							</p>
						</div>
					</div>
				</div>

				<div class="flex gap-2">
					<Button
						variant="solid"
						@click="handleCreate"
						:loading="creating"
						:disabled="!canSubmit"
					>
						{{ __("Create Customer") }}
					</Button>
					<Button variant="subtle" @click="show = false">
						{{ __("Cancel") }}
					</Button>
				</div>
			</div>
		</template>
	</Dialog>
</template>

<script setup>
/**
 * CreateCustomerDialog - Quick customer creation from POS
 *
 * Features:
 * - GSTIN autofill from India Compliance API
 * - Address creation with customer
 * - Country code selector with flag icons
 * - State/Country search dropdowns
 * - Pincode lookup (for India)
 */

import { usePOSPermissions } from "@/composables/usePermissions"
import { useToast } from "@/composables/useToast"
import { useCountriesStore } from "@/stores/countries"
import { logger } from "@/utils/logger"
import { Button, Dialog, createResource, call } from "frappe-ui"
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from "vue"

const log = logger.create("CreateCustomerDialog")

// =============================================================================
// Composables & Stores
// =============================================================================

const countriesStore = useCountriesStore()
const { canCreateCustomer } = usePOSPermissions()
const { showSuccess, showError } = useToast()

// =============================================================================
// Props & Emits
// =============================================================================

const props = defineProps({
	modelValue: Boolean,
	posProfile: String,
	initialName: String,
})

const emit = defineEmits(["update:modelValue", "customer-created"])

// =============================================================================
// State
// =============================================================================

const hasPermission = ref(true)
const checkingPermission = ref(false)
const creating = ref(false)
const selectedCountryCode = ref("+91")
const phoneNumber = ref("")
const showCountryDropdown = ref(false)
const countrySearchQuery = ref("")
const dropdownRef = ref(null)
const countrySearchRef = ref(null)

// State dropdown
const showStateDropdown = ref(false)
const stateSearchQuery = ref("")
const stateDropdownRef = ref(null)

// Country address dropdown
const showCountryAddressDropdown = ref(false)
const countryAddressSearchQuery = ref("India")
const countryAddressDropdownRef = ref(null)

// Pincode suggestions
const pincodeSuggestions = ref([])
const showPincodeSuggestions = ref(false)

// Customer Types and Options
const customerTypes = ref(["Individual", "Company", "Partnership"])
const gstCategories = ref(["Unregistered", "Registered Regular", "Registered Composition", "SEZ", "Overseas", "Deemed Export", "UIN Holders"])
const professions = ref([])
const fetchingGSTIN = ref(false)
const gstinStatus = ref("")

// Indian States for GST
const indianStates = ref([
	"Andaman and Nicobar Islands", "Andhra Pradesh", "Arunachal Pradesh", "Assam",
	"Bihar", "Chandigarh", "Chhattisgarh", "Dadra and Nagar Haveli and Daman and Diu",
	"Delhi", "Goa", "Gujarat", "Haryana", "Himachal Pradesh", "Jammu and Kashmir",
	"Jharkhand", "Karnataka", "Kerala", "Ladakh", "Lakshadweep", "Madhya Pradesh",
	"Maharashtra", "Manipur", "Meghalaya", "Mizoram", "Nagaland", "Odisha",
	"Puducherry", "Punjab", "Rajasthan", "Sikkim", "Tamil Nadu", "Telangana",
	"Tripura", "Uttar Pradesh", "Uttarakhand", "West Bengal"
])

// Customer Data
const customerData = ref({
	customer_name: "",
	customer_type: "Individual",
	custom_profession: "",
	gst_category: "Unregistered",
	email_id: "",
	mobile_no: "",
	gstin: "",
	customer_group: "Customers",
	territory: "All Territories",
})

// Address Data
const addressData = ref({
	address_line1: "",
	address_line2: "",
	city: "",
	state: "",
	country: "India",
	pincode: "",
	is_primary_address: true,
	is_shipping_address: true,
})

// =============================================================================
// Computed
// =============================================================================

const show = computed({
	get: () => props.modelValue,
	set: (val) => emit("update:modelValue", val),
})

const currentCountryCode = computed(() => {
	const country = countriesStore.countries.find((c) => c.isd === selectedCountryCode.value)
	return country?.code.toLowerCase() || "in"
})

const filteredCountries = computed(() => {
	if (!countrySearchQuery.value) return countriesStore.countries
	const query = countrySearchQuery.value.toLowerCase()
	return countriesStore.countries.filter(
		(c) => c.name.toLowerCase().includes(query) || c.isd.includes(query) || c.code.toLowerCase().includes(query)
	)
})

const filteredStates = computed(() => {
	if (!stateSearchQuery.value) return indianStates.value
	const query = stateSearchQuery.value.toLowerCase()
	return indianStates.value.filter((s) => s.toLowerCase().includes(query))
})

const filteredAddressCountries = computed(() => {
	if (!countryAddressSearchQuery.value) return countriesStore.countries
	const query = countryAddressSearchQuery.value.toLowerCase()
	return countriesStore.countries.filter((c) => c.name.toLowerCase().includes(query))
})

const canSubmit = computed(() => {
	return (
		hasPermission.value &&
		customerData.value.customer_name &&
		customerData.value.custom_profession &&
		addressData.value.address_line1 &&
		addressData.value.city &&
		addressData.value.country
	)
})

// =============================================================================
// Country & Phone Methods
// =============================================================================

const handleFlagError = (e) => (e.target.style.display = "none")

const selectCountry = (country) => {
	selectedCountryCode.value = country.isd
	showCountryDropdown.value = false
	countrySearchQuery.value = ""
	updateMobileNumber()
}

const updateMobileNumber = () => {
	customerData.value.mobile_no = phoneNumber.value ? `${selectedCountryCode.value}-${phoneNumber.value}` : ""
}

const handleClickOutside = (event) => {
	// Phone country dropdown
	if (dropdownRef.value && !dropdownRef.value.contains(event.target)) {
		showCountryDropdown.value = false
		countrySearchQuery.value = ""
	}
	// State dropdown
	if (stateDropdownRef.value && !stateDropdownRef.value.contains(event.target)) {
		showStateDropdown.value = false
	}
	// Country address dropdown
	if (countryAddressDropdownRef.value && !countryAddressDropdownRef.value.contains(event.target)) {
		showCountryAddressDropdown.value = false
	}
	// Pincode suggestions
	showPincodeSuggestions.value = false
}

// =============================================================================
// State & Country Selection
// =============================================================================

const selectState = (state) => {
	addressData.value.state = state
	stateSearchQuery.value = state
	showStateDropdown.value = false
}

const selectAddressCountry = (country) => {
	addressData.value.country = country.name
	countryAddressSearchQuery.value = country.name
	showCountryAddressDropdown.value = false
}

// =============================================================================
// Pincode Lookup
// =============================================================================

const onPincodeInput = async () => {
	const pincode = addressData.value.pincode
	if (pincode && pincode.length >= 3 && addressData.value.country === "India") {
		// Simple pincode lookup - in real implementation, call API
		showPincodeSuggestions.value = true
		// For now, just clear suggestions - you can implement actual API lookup
		pincodeSuggestions.value = []
	} else {
		showPincodeSuggestions.value = false
		pincodeSuggestions.value = []
	}
}

const selectPincode = (suggestion) => {
	addressData.value.pincode = suggestion.pincode
	addressData.value.city = suggestion.city
	addressData.value.state = suggestion.state
	stateSearchQuery.value = suggestion.state
	showPincodeSuggestions.value = false
}

// =============================================================================
// GSTIN Autofill Methods
// =============================================================================

const fetchGSTINInfo = async () => {
	const gstin = customerData.value.gstin?.trim()

	if (!gstin || gstin.length !== 15) {
		gstinStatus.value = ""
		return
	}

	fetchingGSTIN.value = true
	gstinStatus.value = "Fetching..."

	try {
		const csrfToken = window.frappe?.csrf_token || window.csrf_token || ""

		const response = await fetch("/api/method/pos_next.api.gstin.get_gstin_info_for_pos", {
			method: "POST",
			headers: {
				"Content-Type": "application/json",
				"X-Frappe-CSRF-Token": csrfToken,
			},
			body: JSON.stringify({ gstin }),
		})

		if (!response.ok) {
			throw new Error(`HTTP ${response.status}: ${response.statusText}`)
		}

		const data = await response.json()

		if (data.exc || data._server_messages) {
			let errorMsg = "Unable to verify GSTIN"
			if (data._server_messages) {
				try {
					const messages = JSON.parse(data._server_messages)
					if (messages.length > 0) {
						const parsedMsg = JSON.parse(messages[0])
						errorMsg = parsedMsg.message || errorMsg
					}
				} catch (e) {
					log.warn("Could not parse server messages", e)
				}
			}
			gstinStatus.value = errorMsg
			return
		}

		if (data.message && !data.message.error) {
			const gstinInfo = data.message

			// Set business name as customer name
			if (gstinInfo.business_name && !customerData.value.customer_name) {
				customerData.value.customer_name = gstinInfo.business_name
			}

			// Set GST Category
			if (gstinInfo.gst_category) {
				customerData.value.gst_category = gstinInfo.gst_category
			}

			// Set customer type based on GSTIN 6th character
			const gstinTypeChar = gstin[5]
			if (gstinTypeChar === "F") {
				customerData.value.customer_type = "Partnership"
			} else if (gstinTypeChar === "C") {
				customerData.value.customer_type = "Company"
			}

			// Set address if available
			if (gstinInfo.permanent_address) {
				const addr = gstinInfo.permanent_address
				if (addr.line1) addressData.value.address_line1 = addr.line1
				if (addr.line2) addressData.value.address_line2 = addr.line2
				if (addr.city) addressData.value.city = addr.city
				if (addr.state) {
					addressData.value.state = addr.state
					stateSearchQuery.value = addr.state
				}
				if (addr.pincode) addressData.value.pincode = addr.pincode
				addressData.value.country = "India"
				countryAddressSearchQuery.value = "India"
			}

			gstinStatus.value = gstinInfo.status ? `Status: ${gstinInfo.status}` : "Details fetched successfully"
			showSuccess(__("GSTIN details fetched successfully"))
		} else {
			gstinStatus.value = data.message?.message || "Invalid GSTIN or unable to fetch details"
		}
	} catch (error) {
		log.error("Error fetching GSTIN info", error)
		gstinStatus.value = error.message || "Network error"
		showError(__("Error: {0}", [error.message]))
	} finally {
		fetchingGSTIN.value = false
	}
}

// =============================================================================
// Load Professions (Custom Field Options)
// =============================================================================

const loadProfessions = async () => {
	try {
		const csrfToken = window.frappe?.csrf_token || window.csrf_token || ""
		// Query Custom Field doctype since custom_profession is a custom field
		const response = await fetch("/api/method/frappe.client.get_list", {
			method: "POST",
			headers: {
				"Content-Type": "application/json",
				"X-Frappe-CSRF-Token": csrfToken,
			},
			body: JSON.stringify({
				doctype: "Custom Field",
				filters: {
					dt: "Customer",
					fieldname: "custom_profession",
				},
				fields: ["options"],
			}),
		})

		if (response.ok) {
			const data = await response.json()
			if (data.message && data.message.length > 0 && data.message[0].options) {
				professions.value = data.message[0].options.split("\n").filter((p) => p.trim())
			}
		}
	} catch (error) {
		log.warn("Could not load professions", error)
		// Use default professions if API fails
		professions.value = ["Doctor", "Accountant", "Business", "CA", "Student"]
	}
}

// =============================================================================
// Create Customer & Address
// =============================================================================

const handleCreate = async () => {
	if (!customerData.value.customer_name) {
		return showError(__("Customer Name is required"))
	}

	if (!customerData.value.custom_profession) {
		return showError(__("Profession is required"))
	}

	if (!addressData.value.address_line1) {
		return showError(__("Address Line 1 is required"))
	}

	if (!addressData.value.city) {
		return showError(__("City is required"))
	}

	creating.value = true

	try {
		// Step 1: Create Customer using frappe.call
		const customerDoc = {
			doctype: "Customer",
			customer_name: customerData.value.customer_name,
			customer_type: customerData.value.customer_type || "Individual",
			customer_group: customerData.value.customer_group || "Customers",
			territory: customerData.value.territory || "All Territories",
			mobile_no: customerData.value.mobile_no || "",
			email_id: customerData.value.email_id || "",
			gstin: customerData.value.gstin || "",
			gst_category: customerData.value.gst_category || "Unregistered",
			custom_profession: customerData.value.custom_profession || "",
		}

		// Create customer using frappe-ui call
		const customerResult = await call("frappe.client.insert", { doc: customerDoc })
		log.info("Customer created", customerResult)

		// Step 2: Create Address linked to Customer
		// Only create address if address_line1 is provided
		if (addressData.value.address_line1) {
			const addressDoc = {
				doctype: "Address",
				address_title: customerData.value.customer_name,
				address_type: "Billing",
				address_line1: addressData.value.address_line1,
				address_line2: addressData.value.address_line2 || "",
				city: addressData.value.city,
				state: addressData.value.state || "",
				country: addressData.value.country || "India",
				pincode: addressData.value.pincode || "",
				is_primary_address: 1,
				is_shipping_address: 1,
				// GST Category is required by India Compliance - use customer's GST category
				gst_category: customerData.value.gst_category || "Unregistered",
				// GSTIN if customer has one
				gstin: customerData.value.gstin || "",
				links: [
					{
						link_doctype: "Customer",
						link_name: customerResult.name,
					},
				],
			}

			try {
				const addressResult = await call("frappe.client.insert", { doc: addressDoc })
				log.info("Address created successfully", addressResult)
			} catch (addrError) {
				log.error("Failed to create address", addrError)
				// Show warning but don't fail the whole operation
				showError(__("Customer created but address could not be saved: {0}", [addrError.message || "Unknown error"]))
			}
		}

		showSuccess(__("Customer {0} created successfully", [customerResult.customer_name]))
		emit("customer-created", customerResult)
		show.value = false
	} catch (error) {
		log.error("Error creating customer", error)
		showError(error.message || __("Failed to create customer"))
	} finally {
		creating.value = false
	}
}

// =============================================================================
// Dialog Lifecycle
// =============================================================================

const loadDialogData = async () => {
	countriesStore.loadCountries()
	loadProfessions()
	checkPermissions()
}

const checkPermissions = async () => {
	checkingPermission.value = true
	try {
		hasPermission.value = await canCreateCustomer()
	} catch (err) {
		log.error("Permission check failed", err)
		hasPermission.value = false
	} finally {
		checkingPermission.value = false
	}
}

const resetForm = () => {
	Object.assign(customerData.value, {
		customer_name: "",
		customer_type: "Individual",
		custom_profession: "",
		gst_category: "Unregistered",
		email_id: "",
		mobile_no: "",
		gstin: "",
		customer_group: "Customers",
		territory: "All Territories",
	})
	Object.assign(addressData.value, {
		address_line1: "",
		address_line2: "",
		city: "",
		state: "",
		country: "India",
		pincode: "",
		is_primary_address: true,
		is_shipping_address: true,
	})
	selectedCountryCode.value = "+91"
	phoneNumber.value = ""
	gstinStatus.value = ""
	stateSearchQuery.value = ""
	countryAddressSearchQuery.value = "India"
}

// =============================================================================
// Watchers
// =============================================================================

watch(
	() => props.initialName,
	(name) => name && (customerData.value.customer_name = name)
)

watch(
	() => customerData.value.mobile_no,
	(value) => {
		if (value?.includes("-")) {
			const [code, ...rest] = value.split("-")
			selectedCountryCode.value = code
			phoneNumber.value = rest.join("-")
		}
	}
)

watch(showCountryDropdown, async (isOpen) => {
	if (isOpen) {
		await nextTick()
		countrySearchRef.value?.focus()
	}
})

watch(
	() => props.modelValue,
	async (isOpen) => {
		show.value = isOpen
		isOpen ? await loadDialogData() : resetForm()
	}
)

watch(show, (val) => emit("update:modelValue", val))

// =============================================================================
// Lifecycle Hooks
// =============================================================================

onMounted(() => {
	loadDialogData()
	document.addEventListener("click", handleClickOutside)
})

onBeforeUnmount(() => {
	document.removeEventListener("click", handleClickOutside)
})
</script>

<style scoped>
.sr-only {
	position: absolute;
	width: 1px;
	height: 1px;
	padding: 0;
	margin: -1px;
	overflow: hidden;
	clip: rect(0, 0, 0, 0);
	white-space: nowrap;
	border-width: 0;
}
</style>
