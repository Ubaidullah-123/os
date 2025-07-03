#include <iostream>
#include <fstream>
#include <string>

#define MAX_PROCESSES 20
#define MAX_RESOURCES 5

using namespace std;

ofstream outfile("ubaid.txt");

struct Process {
    int pid;
    int burst_time;
    int original_bt;
    int priority;
    int arrival_time;
    int waiting_time;
    int turnaround_time;
    int age;
    bool completed;
    bool added;
};

Process proc[MAX_PROCESSES];
int n; // number of processes
int m; // number of resources

int allocation[MAX_PROCESSES][MAX_RESOURCES];
int maximum[MAX_PROCESSES][MAX_RESOURCES];
int need[MAX_PROCESSES][MAX_RESOURCES];
int available[MAX_RESOURCES];

int context_switches = 0;
int deadlocks_detected = 0;
int deadlocks_resolved = 0;
int total_time = 0;

void printGanttChart(int order[], int len) {
    outfile << "\nGantt Chart: ";
    for (int i = 0; i < len; i++) {
        outfile << "|P" << order[i];
    }
    outfile << "|\n";
}

void fcfsScheduling() {
    outfile << "\n--- First Come First Serve Scheduling ---\n";
    int time = 0;
    int gantt[MAX_PROCESSES];

    for (int i = 0; i < n; i++) {
        proc[i].waiting_time = time;
        time += proc[i].burst_time;
        proc[i].turnaround_time = time;
        gantt[i] = proc[i].pid;
        total_time += proc[i].burst_time;
    }

    printGanttChart(gantt, n);
    for (int i = 0; i < n; i++) {
        outfile << "Process " << proc[i].pid
                << ": Waiting = " << proc[i].waiting_time
                << ", Turnaround = " << proc[i].turnaround_time << endl;
    }
}

void roundRobin(int quantum) {
    outfile << "\n--- Round Robin Scheduling ---\n";
    int rem_bt[MAX_PROCESSES];
    for (int i = 0; i < n; i++) rem_bt[i] = proc[i].burst_time;

    int t = 0, done;
    int gantt[100], gidx = 0;

    while (true) {
        done = 1;
        for (int i = 0; i < n; i++) {
            if (rem_bt[i] > 0) {
                done = 0;
                gantt[gidx++] = proc[i].pid;
                context_switches++;
                if (rem_bt[i] > quantum) {
                    t += quantum;
                    rem_bt[i] -= quantum;
                } else {
                    t += rem_bt[i];
                    proc[i].waiting_time = t - proc[i].burst_time;
                    rem_bt[i] = 0;
                }
                total_time = t;
            }
        }
        if (done) break;
    }

    printGanttChart(gantt, gidx);
    for (int i = 0; i < n; i++) {
        proc[i].turnaround_time = proc[i].burst_time + proc[i].waiting_time;
        outfile << "Process " << proc[i].pid
                << ": Waiting = " << proc[i].waiting_time
                << ", Turnaround = " << proc[i].turnaround_time << endl;
    }
}

void priorityScheduling() {
    outfile << "\n--- Priority Scheduling with Aging ---\n";
    int time = 0;
    int completed = 0;
    int gantt[100], gidx = 0;

    while (completed < n) {
        int min_priority = 9999, sel = -1;
        for (int i = 0; i < n; i++) {
            if (!proc[i].completed) {
                int effective_priority = proc[i].priority - proc[i].age;
                if (effective_priority < min_priority) {
                    min_priority = effective_priority;
                    sel = i;
                }
            }
        }

        if (sel != -1) {
            proc[sel].waiting_time = time;
            time += proc[sel].burst_time;
            proc[sel].turnaround_time = time;
            proc[sel].completed = true;
            completed++;
            gantt[gidx++] = proc[sel].pid;
            context_switches++;
            total_time = time;

            for (int j = 0; j < n; j++)
                if (!proc[j].completed) proc[j].age++;
        }
    }

    printGanttChart(gantt, gidx);
    for (int i = 0; i < n; i++) {
        outfile << "Process " << proc[i].pid
                << ": Waiting = " << proc[i].waiting_time
                << ", Turnaround = " << proc[i].turnaround_time << endl;
    }
}

void sjnScheduling() {
    outfile << "\n--- Shortest Job Next Scheduling ---\n";
    int time = 0, completed = 0, gantt[100], gidx = 0;

    while (completed < n) {
        int min_bt = 9999, sel = -1;
        for (int i = 0; i < n; i++) {
            if (!proc[i].completed && proc[i].burst_time < min_bt) {
                min_bt = proc[i].burst_time;
                sel = i;
            }
        }
        if (sel != -1) {
            proc[sel].waiting_time = time;
            time += proc[sel].burst_time;
            proc[sel].turnaround_time = time;
            proc[sel].completed = true;
            completed++;
            gantt[gidx++] = proc[sel].pid;
            context_switches++;
            total_time = time;
        }
    }

    printGanttChart(gantt, gidx);
    for (int i = 0; i < n; i++) {
        outfile << "Process " << proc[i].pid
                << ": Waiting = " << proc[i].waiting_time
                << ", Turnaround = " << proc[i].turnaround_time << endl;
    }
}

void inputResources() {
    cout << "Enter number of resource types: ";
    cin >> m;
    cout << "Enter available resources: ";
    for (int i = 0; i < m; i++) cin >> available[i];

    for (int i = 0; i < n; i++) {
        cout << "Process " << i << ":\n";
        for (int j = 0; j < m; j++) {
            cout << "  Allocation for R" << j << ": ";
            cin >> allocation[i][j];
        }
        for (int j = 0; j < m; j++) {
            cout << "  Maximum need for R" << j << ": ";
            cin >> maximum[i][j];
        }
    }
}

void calculateNeed() {
    for (int i = 0; i < n; i++)
        for (int j = 0; j < m; j++)
            need[i][j] = maximum[i][j] - allocation[i][j];
}

bool isSafe() {
    int work[MAX_RESOURCES], finish[MAX_PROCESSES] = {0};
    for (int i = 0; i < m; i++) work[i] = available[i];

    int count = 0;
    while (count < n) {
        bool found = false;
        for (int p = 0; p < n; p++) {
            if (!finish[p]) {
                int j;
                for (j = 0; j < m; j++)
                    if (need[p][j] > work[j]) break;

                if (j == m) {
                    for (int k = 0; k < m; k++)
                        work[k] += allocation[p][k];
                    finish[p] = 1;
                    found = true;
                    count++;
                }
            }
        }
        if (!found) return false;
    }
    return true;
}

void resolveDeadlock() {
    deadlocks_detected++;
    outfile << "\nDEADLOCK DETECTED!\nAttempting to resolve by killing one process...\n";
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < m; j++) {
            available[j] += allocation[i][j];
            allocation[i][j] = 0;
            maximum[i][j] = 0;
        }
        outfile << "Process " << i << " killed. Resources released.\n";
        break;
    }
    calculateNeed();
    if (isSafe()) {
        outfile << "System is now SAFE after resolution.\n";
        deadlocks_resolved++;
    }
}

void printRAG() {
    outfile << "\n--- Resource Allocation Graph (RAG) ---\n";
    for (int i = 0; i < n; i++) {
        outfile << "P" << i << " -> ";
        for (int j = 0; j < m; j++) {
            if (allocation[i][j] > 0)
                outfile << "R" << j << "(A=" << allocation[i][j] << ") ";
        }
        outfile << "\n";
    }
}

void addProcessDynamically() {
    char ch;
    cout << "\nDo you want to add a process dynamically? (y/n): ";
    cin >> ch;
    while (ch == 'y' && n < MAX_PROCESSES) {
        proc[n].pid = n;
        cout << "Enter burst time of P" << n << ": ";
        cin >> proc[n].burst_time;
        proc[n].original_bt = proc[n].burst_time;
        cout << "Enter priority of P" << n << " (lower is higher): ";
        cin >> proc[n].priority;
        proc[n].arrival_time = 0;
        proc[n].waiting_time = 0;
        proc[n].turnaround_time = 0;
        proc[n].age = 0;
        proc[n].completed = false;
        proc[n].added = true;
        n++;
        cout << "Add another process? (y/n): ";
        cin >> ch;
    }
}

void performanceSummary() {
    float avg_wt = 0, avg_tt = 0;
    for (int i = 0; i < n; i++) {
        avg_wt += proc[i].waiting_time;
        avg_tt += proc[i].turnaround_time;
    }
    avg_wt /= n;
    avg_tt /= n;
    float utilization = ((float)total_time / (total_time + context_switches)) * 100;

    outfile << "\n--- Performance Summary ---\n";
    outfile << "Average Waiting Time: " << avg_wt << "\n";
    outfile << "Average Turnaround Time: " << avg_tt << "\n";
    outfile << "Context Switches: " << context_switches << "\n";
    outfile << "CPU Utilization: " << utilization << "%\n";
    outfile << "Deadlocks Detected: " << deadlocks_detected
            << ", Resolved: " << deadlocks_resolved << "\n";
}

void autoSwitchScheduling() {
    float avg_bt = 0;
    for (int i = 0; i < n; i++) avg_bt += proc[i].burst_time;
    avg_bt /= n;

    if (avg_bt <= 5) {
        fcfsScheduling();
    } else if (avg_bt <= 10) {
        roundRobin(3);
    } else if (avg_bt <= 15) {
        sjnScheduling();
    } else {
        priorityScheduling();
    }

    performanceSummary();
}

int main() {
    cout << "Enter number of initial processes: ";
    cin >> n;
    for (int i = 0; i < n; i++) {
        proc[i].pid = i;
        cout << "Enter burst time of P" << i << ": ";
        cin >> proc[i].burst_time;
        proc[i].original_bt = proc[i].burst_time;
        cout << "Enter priority of P" << i << " (lower is higher): ";
        cin >> proc[i].priority;
        proc[i].arrival_time = 0;
        proc[i].waiting_time = 0;
        proc[i].turnaround_time = 0;
        proc[i].age = 0;
        proc[i].completed = false;
        proc[i].added = false;
    }

    inputResources();
    calculateNeed();
    printRAG();

    if (!isSafe()) {
        resolveDeadlock();
    } else {
        outfile << "System is in a safe state.\n";
    }

    addProcessDynamically();
    autoSwitchScheduling();

    outfile.close();
    cout << "Simulation completed. Check 'ubaid.txt'.\n";
    return 0;
}
