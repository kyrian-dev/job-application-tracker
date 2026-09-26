<script setup>
import { ref, watch, computed, h } from 'vue'

const company = ref('')
const position = ref('')
const applicationDate = ref('')
const jobLink = ref('')
const editingId = ref(null)

const search = ref('')
const statusFilter = ref('All')

const savedApplications = localStorage.getItem('applications')
const applications = ref(
  savedApplications ? JSON.parse(savedApplications) : []
)

watch(
  applications,
  (updatedApplications) => {
    localStorage.setItem(
      'applications',
      JSON.stringify(updatedApplications)
    )
  },
  { deep: true }
)

function addApplication() {
  const companyName = company.value.trim()
  const jobTitle = position.value.trim()

  if (!companyName || !jobTitle) return

  const details = {
    company: companyName,
    position: jobTitle,
    date: applicationDate.value,
    jobLink: jobLink.value.trim(),
  }

  if (editingId.value !== null) {
    const application = applications.value.find(
      (item) => item.id === editingId.value
    )

    if (!application) {
      resetForm()
      return
    }

    Object.assign(application, details)
  } else {
    applications.value.push({
      id: crypto.randomUUID(),
      ...details,
      status: 'Applied',
    })
  }

  resetForm()
}

function editApplication(application) {
  editingId.value = application.id
  company.value = application.company
  position.value = application.position
  applicationDate.value = application.date || ''
  jobLink.value = application.jobLink || ''
}

function resetForm() {
  editingId.value = null
  company.value = ''
  position.value = ''
  applicationDate.value = ''
  jobLink.value = ''
}

function deleteApplication(id) {
  const application = applications.value.find(
    (item) => item.id === id
  )

  if (!application) return

  const confirmed = window.confirm(
    `Delete the application for ${application.position} at ${application.company}?`
  )

  if (!confirmed) return

  applications.value = applications.value.filter(
    (item) => item.id !== id
  )

  if (editingId.value === id) {
    resetForm()
  }
}

const filteredApplications = computed(() => {
  const searchText = search.value.trim().toLowerCase()

  return applications.value
    .filter((application) => {
      const matchesCompany = application.company
        .toLowerCase()
        .includes(searchText)

      const matchesStatus =
        statusFilter.value === 'All' ||
        application.status === statusFilter.value

      return matchesCompany && matchesStatus
    })
    .sort((a, b) => (b.date || '').localeCompare(a.date || ''))
})

const applicationStats = computed(() => ({
  total: applications.value.length,
  applied: applications.value.filter(
    (app) => app.status === 'Applied'
  ).length,
  interviews: applications.value.filter(
    (app) => app.status === 'Interview'
  ).length,
  offers: applications.value.filter(
    (app) => app.status === 'Offer'
  ).length,
}))

</script>

<template>
  <main>
    <h1>Job Application Tracker</h1>
    <p>Keep track of your applications and interviews.</p>
    <div class="stats">
  <div class="stat-card">
    <strong>{{ applicationStats.total }}</strong>
    <span>Total applications</span>
  </div>

  <div class="stat-card">
    <strong>{{ applicationStats.applied }}</strong>
    <span>Applied</span>
  </div>

  <div class="stat-card">
    <strong>{{ applicationStats.interviews }}</strong>
    <span>Interviews</span>
  </div>

  <div class="stat-card">
    <strong>{{ applicationStats.offers }}</strong>
    <span>Offers</span>
  </div>
</div>

    <form @submit.prevent="addApplication">
      <div class="field">
        <label for="company">Company name</label>
        <input
          id="company"
          v-model="company"
          placeholder="Berg Point"
          required
        />
      </div>

      <div class="field">
        <label for="position">Job position</label>
        <input
          id="position"
          v-model="position"
          placeholder="Junior Web Developer"
          required
        />
      </div>

      <div class="field">
        <label for="application-date">Application date</label>
        <input
          id="application-date"
          v-model="applicationDate"
          type="date"
          required
        />
      </div>

      <div class="field">
        <label for="job-link">Job link (optional)</label>
        <input
          id="job-link"
          v-model="jobLink"
          type="url"
          placeholder="https://example.com/jobs/junior-developer"
        />
      </div>

      <button type="submit">
        {{ editingId !== null ? 'Save changes' : 'Add application' }}
      </button>

      <button
        v-if="editingId !== null"
        type="button"
        @click="resetForm"
      >
        Cancel editing
      </button>
    </form>

    <section aria-labelledby="applications-heading">
      <h2 id="applications-heading">My applications</h2>

      <p class="application-count" role="status">
        Showing {{ filteredApplications.length }}
        of {{ applications.length }} applications
      </p>

      <div class="field">
        <label for="search">Search by company</label>
        <input
          id="search"
          v-model="search"
          type="search"
          placeholder="Type a company name"
        />
      </div>

      <div class="field">
        <label for="status-filter">Filter by status</label>
        <select id="status-filter" v-model="statusFilter">
          <option>All</option>
          <option>Applied</option>
          <option>Interview</option>
          <option>Offer</option>
          <option>Rejected</option>
        </select>
      </div>

      <p v-if="applications.length === 0">
        No applications yet. Add your first one above.
      </p>

      <p v-else-if="filteredApplications.length === 0">
        No applications match your search and filters.
      </p>

      <ul v-else class="application-list">
        <li
          v-for="application in filteredApplications"
          :key="application.id"
          class="application-card"
        >
          <h3>{{ application.position }}</h3>
          <p>{{ application.company }}</p>

          <p v-if="application.date">
            Applied on: {{ application.date }}
          </p>

          <a
            v-if="
              application.jobLink &&
              /^https?:\/\//i.test(application.jobLink)
            "
            :href="application.jobLink"
            target="_blank"
            rel="noopener noreferrer"
          >
            View job advertisement
          </a>

          <label :for="'status-' + application.id">Status</label>
          <select
            :id="'status-' + application.id"
            v-model="application.status"
          >
            <option>Applied</option>
            <option>Interview</option>
            <option>Offer</option>
            <option>Rejected</option>
          </select>

          <hr>

          <button
            type="button"
            @click="editApplication(application)"
          >
            Edit application
          </button>

          <button
            type="button"
            class="delete-button"
            @click="deleteApplication(application.id)"
          >
            Delete application
          </button>
        </li>
      </ul>
    </section>
  </main>
</template>