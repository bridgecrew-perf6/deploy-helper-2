<template>
  <section class="container mx-auto">
    <p v-if="loading">Loading...</p>
    <table class="table-auto" v-if="!loading">
      <thead>
        <tr>
          <th class="px-4 py-2">Name</th>
          <th class="px-4 py-2">Last Change</th>
          <th class="px-4 py-2">Open PR's</th>
          <th class="px-4 py-2">Workflow history</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="repo of repos" :key="repo.id">
          <td class="border px-4 py-2">
            <a
              :href="repo.html_url"
              class="font-medium text-blue-500 underline hover:text-blue-700"
              >{{ repo.name }}</a
            >
          </td>
          <td class="border px-4 py-2">{{ repo.pushed_at }}</td>
          <td class="border px-4 py-2">
            <span
              class="inline-flex items-center justify-center px-2 py-1 text-xs font-bold leading-none text-indigo-100 rounded"
              :class="
                repo.open_issues_count > 3 ? 'bg-pink-600' : 'bg-indigo-700'
              "
              >{{ repo.open_issues_count }}</span
            >
          </td>
          <td class="border px-4 py-2" v-html="repo.workflow"></td>
        </tr>
      </tbody>
    </table>

    <div class="flex items-center justify-center p-3" v-if="!loading">
      <button
        @click="prev"
        class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded"
      >
        Prev</button
      >&nbsp;
      <button
        @click="next"
        class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded"
      >
        Next
      </button>
    </div>
  </section>
</template>

<script lang="ts">
import Vue from "vue";
import { Octokit } from "@octokit/core";

const octokit = new Octokit({
  auth: "",
});

export default Vue.extend({
  name: "IndexPage",

  data() {
    return {
      owner: "SPOXX",
      repos: [] as any,
      page: 1,
      loading: false,
    };
  },

  created() {
    this.fetchRepos();
  },

  computed: {},

  methods: {
    next() {
      this.page += 1;
      this.fetchRepos();
    },
    prev() {
      this.page -= 1;
      this.fetchRepos();
    },
    async fetchRepos() {
      this.loading = true;
      const response = await octokit.request(
        `GET /orgs/${this.owner}/repos?page=${this.page}&per_page=8&sort=pushed`,
        {
          org: "octokit",
          type: "private",
        }
      );
      response.data.forEach(async (repo: any) => {
        this.repos.push({
          ...repo,
          workflow: await this.fetchWorkflowStatus(repo.name),
        });
      });
      this.loading = false;
    },

    branchColour(branch: string) {
      switch (branch) {
        case "develop":
          return "bg-green-200";
        case "qa":
          return "bg-yellow-200";
        case "acc":
          return "bg-red-300";
        default:
          return "bg-gray-200";
      }
    },

    statusColour(status: string) {
      switch (status) {
        case "in_progress":
          return "bg-gray-700";
        case "completed":
          return "bg-green-800";
        default:
          return "bg-red-400";
      }
    },

    findImpactBranches(branch: string) {
      switch (branch) {
        case "develop":
        case "qa":
        case "acc":
        case "release":
        case "test":
        case "master":
          return true;
        default:
          return false;
      }
    },

    printAdditionalInfo(repo: any, workflow: any) {
      if (repo === "k8s-gitops") return workflow.head_commit.message;

      return workflow.name;
    },

    async fetchWorkflowStatus(repo: string) {
      try {
        const response = await octokit.request(
          `GET /repos/SPOXX/${repo}/actions/runs?per_page=8`
        );
        console.log(response);

        let output = "";
        response.data.workflow_runs
          .filter((workflow: any) =>
            this.findImpactBranches(workflow.head_branch)
          )
          .forEach((workflow: any) => {
            output += `<div class="p-0.15">
            <a href='${
              workflow.html_url
            }' target='_blank' class='font-medium text-blue-500 underline hover:text-blue-700'>${this.printAdditionalInfo(
              repo,
              workflow
            )}</a> |
            <span class="inline-flex ${this.statusColour(
              workflow.status
            )} text-white rounded-full h-6 px-3 justify-center items-center text-">${
              workflow.status
            }</span> |
            <span class="inline-flex ${this.branchColour(
              workflow.head_branch
            )} text-dark rounded-full h-6 px-3 justify-center items-center text-">${
              workflow.head_branch
            }</span> |
            ${new Date(workflow.updated_at).toLocaleString()}
          </div><br />`;
          });

        return output;
      } catch (e) {
        console.warn(repo, e);
        return "Could not find actions";
      }
    },
  },
});
</script>
