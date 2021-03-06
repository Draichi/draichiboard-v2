<template>
  <div v-if="!loading" class="row full-height">
    <div class="col-auto column">
      <q-card class="q-ml-md q-mt-md full-height">
        <div>Total Contributions</div>
        <DoughnutChart :data="dataPie" />
        <div>Total Contributions per year</div>
        <BarChart :data="dataPie2" />
      </q-card>
    </div>
    <div class="col">
      <q-card class="q-ma-md">
        <div class="main-chart__container">
          <h4 class="main-chart__subtitle">Total contributions</h4>
          <h2 class="main-chart__title">Commits</h2>
          <LineChart :data="sanitazedData" style="height: 35vh" />
        </div>
      </q-card>

      <div class="q-mx-md row justify-around items-top">
        <q-card class="col column flex items-center justify-center" style="height: 40vh">
          <div>Total Repositories</div>
          <div class="small-dognut-chart__container">
            <DoughnutChart :data="dataPie3" />
          </div>
        </q-card>
        <!-- make it green like the line one-->
        <q-card
          class="q-mx-md col flex justify-center items-center relative-position"
          style="height: 40vh"
        >
          <div>Repos created per year</div>
          <HorizontalBar :data="dataRepos" />
        </q-card>
        <q-card class="col flex justify-center items-center" style="height: 40vh">
          <div
            class="q-pa-lg"
            style="max-width: 20vw; line-height:2; letter-spacing: .08rem;
                font-weight: 100;"
          >
            <p>total commits: {{ this.totalCommitContributions }}</p>
          </div>
        </q-card>
      </div>
    </div>
  </div>
  <div v-else class="row justify-center items-center window-height">
    <q-spinner-cube size="200" color="secondary" />
  </div>
</template>

<script>
import LineChart from 'components/LineChart';
import DoughnutChart from 'components/DoughnutChart';
import BarChart from 'components/BarChart';

const today = new Date().toISOString();

const getWeeklyContributions = (week) => {
  const weekContributionsArray = week.contributionDays.reduce(
    (acc, val) => acc.concat(val.contributionCount),
    [],
  );
  const weekContributions = weekContributionsArray.reduce((a, b) => a + b, 0);
  return weekContributions;
};

const monthNames = [
  'Jan',
  'Feb',
  'Mar',
  'Apr',
  'May',
  'Jun',
  'Jul',
  'Aug',
  'Sep',
  'Oct',
  'Nov',
  'Dec',
];

const getContributionsByMonth = (array, size) => {
  if (array.length <= size) {
    return [array];
  }
  return [
    array.slice(0, size),
    ...getContributionsByMonth(array.slice(size), size),
  ];
};

const sanitazeData = (contributionCalendarWeeks) => {
  const weeklyContributions = contributionCalendarWeeks
    .map(getWeeklyContributions)
    .flat();
  const montlyContributionsArray = getContributionsByMonth(
    weeklyContributions,
    4,
  );
  const montlyContributions = montlyContributionsArray.map((item) => item.reduce((a, b) => a + b));
  return montlyContributions;
};

const sanitazeLabels = (contributionCalendarWeeks, yearLabel) => {
  const weeklyContributions = contributionCalendarWeeks
    .map((item) => {
      const { length } = item.contributionDays;
      const { date } = item.contributionDays[length - 1];
      const monthNumber = new Date(date).getMonth();
      const formatedDate = `${monthNames[monthNumber]} ${yearLabel}`;
      return formatedDate;
    })
    .flat();
  const montlyContributionsArray = getContributionsByMonth(
    weeklyContributions,
    4,
  );
  const montlyContributions = montlyContributionsArray.map(
    (item) => item[item.length - 1],
  );
  return montlyContributions;
};

export default {
  name: 'PageIndex',
  components: {
    LineChart,
    DoughnutChart,
    BarChart,
    HorizontalBar: () => import('components/HorizontalBarChart'),
  },
  data() {
    return {
      sanitazedData: null,
      data: [],
      labels: [],
      loading: true,
      dataPie: null,
      totalIssueContributions: null,
      totalPullRequestContributions: null,
      totalCommitContributions: null,
      totalPullRequestReviewContributions: null,
      totalRepositoriesWithContributedCommits: null,
      totalRepositoriesWithContributedIssues: null,
      tpr: null,
      tprr: null,
      years: [
        {
          from: '2020-01-01T00:00:00',
          to: today,
          label: '20',
        },
        {
          from: '2019-01-01T00:00:00',
          to: '2019-12-01T00:00:00',
          label: '19',
        },
        {
          from: '2018-01-01T00:00:00',
          to: '2018-12-01T00:00:00',
          label: '18',
        },
        {
          from: '2017-11-01T00:00:00',
          to: '2017-12-01T00:00:00',
          label: '17',
        },
      ],
    };
  },
  mounted() {
    this.getYearlyContributions();
  },
  methods: {
    async getContributions(token, username, year) {
      const headers = {
        Authorization: `bearer ${token}`,
      };
      const body = {
        // https://developer.github.com/v4/object/contributionscollection/
        query: `query {
            user(login: "${username}") {
              contributionsCollection(from: "${year.from}", to: "${year.to}") {
                totalCommitContributions
                totalIssueContributions
                totalPullRequestContributions
                totalPullRequestReviewContributions
                totalRepositoriesWithContributedCommits
                totalRepositoriesWithContributedIssues
                totalRepositoriesWithContributedPullRequests
                totalRepositoriesWithContributedPullRequestReviews
                totalRepositoryContributions
                contributionCalendar {
                  totalContributions
                  weeks {
                    contributionDays {
                      contributionCount
                      date
                    }
                  }
                }
              }
            }
          }`,
      };

      const response = await fetch('https://api.github.com/graphql', {
        method: 'POST',
        body: JSON.stringify(body),
        headers,
      });
      const dataaa = await response.json();
      const {
        totalIssueContributions,
        totalPullRequestContributions,
        totalCommitContributions,
        totalPullRequestReviewContributions,
        totalRepositoriesWithContributedCommits,
        totalRepositoriesWithContributedIssues,
        totalRepositoriesWithContributedPullRequests,
        totalRepositoriesWithContributedPullRequestReviews,
        totalRepositoryContributions,
        contributionCalendar,
      } = dataaa.data.user.contributionsCollection;
      this.totalIssueContributions += totalIssueContributions;
      this.totalPullRequestContributions += totalPullRequestContributions;
      this.totalCommitContributions += totalCommitContributions;
      this.totalPullRequestReviewContributions += totalPullRequestReviewContributions;
      this.totalRepositoriesWithContributedCommits += totalRepositoriesWithContributedCommits;
      this.totalRepositoriesWithContributedIssues += totalRepositoriesWithContributedIssues;
      this.tpr += totalRepositoriesWithContributedPullRequests;
      this.tprr += totalRepositoriesWithContributedPullRequestReviews;
      const contributionCalendarWeeks = contributionCalendar.weeks;
      return {
        data: await sanitazeData(contributionCalendarWeeks),
        label: await sanitazeLabels(contributionCalendarWeeks, year.label),
        totalIssueContributions,
        totalCommitContributions,
        totalPullRequestContributions,
        totalPullRequestReviewContributions,
        totalRepositoriesWithContributedCommits,
        totalRepositoriesWithContributedIssues,
        totalRepositoryContributions,
      };
    },
    async getYearlyContributions() {
      const data20 = await this.getContributions(
        process.env.GTOKEN,
        'Draichi',
        this.years[0],
      );
      const data19 = await this.getContributions(
        process.env.GTOKEN,
        'Draichi',
        this.years[1],
      );
      const data18 = await this.getContributions(
        process.env.GTOKEN,
        'Draichi',
        this.years[2],
      );
      const data17 = await this.getContributions(
        process.env.GTOKEN,
        'Draichi',
        this.years[3],
      );
      this.sanitazedData = {
        labels: [
          ...data17.label,
          ...data18.label,
          ...data19.label,
          ...data20.label,
        ],
        datasets: [
          {
            label: 'Total contributions',
            pointBackgroundColor: 'white',
            borderWidth: 1,
            borderColor: '#911215',
            data: [
              ...data17.data,
              ...data18.data,
              ...data19.data,
              ...data20.data,
            ],
          },
        ],
      };
      this.dataPie = {
        labels: ['Issues', 'Pull requests', 'Reviews'],
        datasets: [
          {
            backgroundColor: [
              'rgba(120, 74, 211, 0.29)',
              'rgba(74, 211, 150, 0.29)',
              'rgba(47, 12, 150, 0.29)',
            ],
            data: [
              this.totalIssueContributions,
              this.totalPullRequestContributions,
              this.totalPullRequestReviewContributions,
            ],
          },
        ],
      };
      this.dataPie2 = {
        labels: ['2017', '2018', '2019', '2020'],
        datasets: [
          {
            fill: true,
            label: 'Total contributions',
            borderColor: '#1d8cf8',
            borderWidth: 2,
            borderDash: [],
            borderDashOffset: 0.0,
            data: [
              data17.totalCommitContributions,
              data18.totalCommitContributions,
              data19.totalCommitContributions,
              data20.totalCommitContributions,
            ],
          },
        ],
      };
      this.dataRepos = {
        labels: ['2017', '2018', '2019', '2020'],
        datasets: [
          {
            fill: true,
            label: 'Repos created',
            borderColor: '#1d8cf8',
            borderWidth: 2,
            borderDash: [],
            borderDashOffset: 0.0,
            data: [
              data17.totalRepositoryContributions,
              data18.totalRepositoryContributions,
              data19.totalRepositoryContributions,
              data20.totalRepositoryContributions,
            ],
          },
        ],
      };
      this.dataPie3 = {
        labels: ['With PRs', 'With Reviews', 'With Commits', 'With Issues'],
        datasets: [
          {
            backgroundColor: [
              'rgba(120, 74, 211, 0.29)',
              'rgba(74, 211, 150, 0.29)',
              'rgba(47, 12, 150, 0.29)',
              'rgba(0, 12, 150, 0.29)',
            ],
            data: [
              this.tpr,
              this.tprr,
              this.totalRepositoriesWithContributedCommits,
              this.totalRepositoriesWithContributedIssues,
            ],
          },
        ],
      };
      this.loading = false;
    },
  },
};
</script>

<style lang="scss">
.main-chart {
  &__container {
    padding: 1rem;
  }
  &__subtitle {
    margin-top: 0;
    color: rgb(168, 161, 149);
    margin-bottom: 0;
    font-size: 0.75rem;
    line-height: 1.2;
  }
  &__title {
    margin-top: 0;
    color: rgb(232, 230, 227);
    margin-bottom: 0.75rem;
    font-weight: 100;
    line-height: 1.2;
    font-size: 1.68rem;
  }
}
.small-dognut-chart__container {
  width: 300px;
}

canvas#horizontalbar-chart {
  width: 90% !important;
  // height: 90% !important;
}
</style>
