---
share_link: https://share.note.sx/aruxuqa1#kIBJqCqTh96AMVlA2Xgjn1pOHc7eLEVCNx4KS+xTHAc
share_updated: 2025-05-13T05:39:00+03:00
---

```c
#include <mpi.h>
#include <iostream>
#include "quick_search.h"
#include "prime_finder.h"
#include "radixSort.h"
#include "bitonicSort.h"

using namespace std;

void display_menu(int rank) {
    if (rank == 0) {
        cout << "\n============================================\n";
        cout << "Welcome to Parallel Algorithm Simulation with MPI\n";
        cout << "============================================\n";
        cout << "Please choose an algorithm to execute:\n";
        cout << "01 - Quick Search\n";
        cout << "02 - Prime Number Finding\n";
        cout << "03 - Radix Sort\n";
        cout << "04 - Sample Sort\n";
        cout << "00 - Exit\n";
        cout << "Enter your choice: ";
        cout.flush();  // Ensure prompt is displayed
    }
}

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);

    int rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    int keepRunning = 1;

    while (keepRunning) {
        display_menu(rank);

        int choice = 0;
        if (rank == 0) {
            cin >> choice;
            cin.clear();
            cin.ignore(numeric_limits<streamsize>::max(), '\n');
        }

        MPI_Bcast(&choice, 1, MPI_INT, 0, MPI_COMM_WORLD);

        switch (choice) {
        case 0:
            keepRunning = 0;
            break;
        case 1:
            if (rank == 0) cout << "Quick Search Selected\n";
            quick_search();
            break;
        case 2:
            if (rank == 0) cout << "Prime Finding Selected\n";
            prime_finder();
            break;
        case 3:
            if (rank == 0) cout << "Radix Sort Selected\n";
            radix_sort();
            break;
        case 4:
            if (rank == 0) cout << "Sample Sort Selected\n";
            break;
        default:
            if (rank == 0) cout << "Invalid choice. Please try again.\n";
        }

        if (keepRunning && choice >= 0 && choice <= 5 && rank == 0) {
            char again;
            cout << "Run another algorithm? (Y/N): ";
            cin >> again;
            keepRunning = (again == 'Y' || again == 'y');
            cin.clear();
            cin.ignore(numeric_limits<streamsize>::max(), '\n');
        }

        MPI_Bcast(&keepRunning, 1, MPI_INT, 0, MPI_COMM_WORLD);
    }

    if (rank == 0) {
        cout << "Exiting program. Goodbye!\n";
    }

    MPI_Finalize();
    return 0;
}

```


```c

#include "radixSort.h" // Assuming this contains read_input_file, etc.
#include "utils.h"     // Make sure this is included if read_input_file is here
#include <mpi.h>
#include <iostream>
#include <vector>       // Use vector consistently
#include <algorithm>
#include <fstream>
#include <stdexcept>
#include <cmath>        // For ceil
#include <numeric>      // For std::iota potentially, or prefix sums if needed
#include <limits> // Required for numeric_limits

using namespace std;


// Define the MinMax struct to hold min and max values
struct MinMax {
    int min_val = numeric_limits<int>::max();
    int max_val = numeric_limits<int>::min();
};



MinMax get_min_max(const vector<int>& arr) { 
    MinMax result;
	// Check if the array is empty
    if (arr.empty()) {
        result.min_val = 0;
        result.max_val = 0;
        return result;
    }
	// Initialize min and max with the first element
    result.min_val = arr[0];
    result.max_val = arr[0];
    // Use size_t for loop counter matching arr.size() type
    for (size_t i = 1; i < arr.size(); ++i) {
        if (arr[i] < result.min_val) {
            result.min_val = arr[i];
        }
        if (arr[i] > result.max_val) {
            result.max_val = arr[i];
        }
    }
    return result;
}


void counting_sort_neg(vector<int>& arr, long long exp) {
    if (arr.empty() || exp <= 0) return;

    const int RANGE = 19; // -9 to 9
    const int OFFSET = 9;
    size_t n = arr.size();
    vector<int> output(n);
    vector<int> count(RANGE, 0);

    // Count digit occurrences
    for (int num : arr) {
        int digit = (num / exp) % 10;
        count[digit + OFFSET]++;
    }

    // Cumulative count
    for (int i = 1; i < RANGE; ++i) {
        count[i] += count[i - 1];
    }

    // Build output array (in reverse for stability)
    for (size_t i = n; i-- > 0; ) {
        int digit = (arr[i] / exp) % 10;
        int& pos = count[digit + OFFSET];
        output[--pos] = arr[i];
    }

    arr = move(output);
}


void k_way_merge(const vector<int>& data,
    const vector<int>& counts, // MPI counts are int
    const vector<int>& displs, // MPI displacements are int
    int k,
    vector<int>& sorted_output)
{
    // Use size_t for total size, cast data.size()
    size_t n_total = data.size(); // data.size() is size_t
    if (n_total == 0) {
        sorted_output.clear(); // Ensure output is empty
        return;
    }

    sorted_output.resize(n_total); // resize takes size_t

    // Use size_t for indices within potentially large merged array
    vector<size_t> current_indices(k, 0);
    // End indices are counts, which are int from MPI. Store as size_t for comparison.
    vector<size_t> end_indices(k);
    for (int i = 0; i < k; ++i) {
        // Be careful if counts[i] could be negative (shouldn't be from MPI_Gather)
        end_indices[i] = (counts[i] > 0) ? static_cast<size_t>(counts[i]) : 0;
    }

    for (size_t out_idx = 0; out_idx < n_total; ++out_idx) {
        int min_val = numeric_limits<int>::max();
        int min_k = -1;

        for (int i = 0; i < k; ++i) {
            // Compare size_t indices
            if (current_indices[i] < end_indices[i]) {
                // Calculate global index: displs[i] is int, current_indices[i] is size_t
                // Cast displs[i] to size_t before adding to avoid potential issues
                // if displs[i] is large positive and current_indices is also large.
                // Check for negative displacement just in case (shouldn't happen).
                size_t global_idx;
                if (displs[i] >= 0) {
                    global_idx = static_cast<size_t>(displs[i]) + current_indices[i];
                }
                else {
                    cerr << "Error: Negative displacement encountered in k-way merge." << endl;
                    // Handle error - maybe skip this subarray?
                    continue;
                }

                // **** Bounds Check ****: Ensure global_idx is valid for data
                if (global_idx < data.size()) {
                    if (data[global_idx] <= min_val) {
                        min_val = data[global_idx];
                        min_k = i;
                    }
                }
                else {
                    cerr << "Error: Calculated global index " << global_idx << " out of bounds (size=" << data.size() << ") in k-way merge." << endl;
                    // This indicates an issue with counts/displs calculation earlier
                    // We might want to skip this subarray or abort
                    current_indices[i] = end_indices[i]; // Mark as exhausted to avoid re-checking
                }
            }
        }

        if (min_k != -1) {
            sorted_output[out_idx] = min_val;
            current_indices[min_k]++;
        }
        else {
            // Only an error if out_idx < n_total
            if (out_idx < n_total) {
                cerr << "Error during k-way merge: Could not find minimum element for output index " << out_idx << "." << endl;
            }
            // If this happens, likely means counts/displs were wrong, or n_total calculation differed.
            break;
        }
    }
 
}


void radix_sort() {
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    double start_time = 0.0, end_time = 0.0;
    vector<int> data;
    int global_n = 0;

    if (rank == 0) {
        try {
            string filename;
            cout << "Enter input file path: ";
            cin >> filename;
            data = read_input_file(filename);
            if (data.size() > static_cast<size_t>(numeric_limits<int>::max())) {
                cerr << "Too much data to handle with MPI. Aborting.\n";
                global_n = -1;
            }
            else {
                global_n = static_cast<int>(data.size());
            }
        }
        catch (const exception& e) {
            cerr << "Error reading file: " << e.what() << endl;
            global_n = -1;
        }
    }


    MPI_Bcast(&global_n, 1, MPI_INT, 0, MPI_COMM_WORLD);
    if (global_n <= 0) {
        if (rank == 0) cout << "No valid data. Exiting.\n";
        return;
    }

    start_time = MPI_Wtime();

    // Compute distribution info
    int chunk_size = global_n / size;
    int remainder = global_n % size;
    int local_n = chunk_size + (rank < remainder ? 1 : 0);

    vector<int> local_data(local_n);
    vector<int> sendcounts, displs;

    if (rank == 0) {
        sendcounts.resize(size);
        displs.resize(size);
        int offset = 0;
        for (int i = 0; i < size; ++i) {
            sendcounts[i] = chunk_size + (i < remainder ? 1 : 0);
            displs[i] = offset;
            offset += sendcounts[i];
        }
    }

    MPI_Scatterv(rank == 0 ? data.data() : nullptr,
        rank == 0 ? sendcounts.data() : nullptr,
        rank == 0 ? displs.data() : nullptr,
        MPI_INT,
        local_data.data(), local_n,
        MPI_INT,
        0, MPI_COMM_WORLD);

    if (!local_data.empty()) {
        MinMax mm = get_min_max(local_data);
        int global_min, global_max;
        MPI_Allreduce(&mm.min_val, &global_min, 1, MPI_INT, MPI_MIN, MPI_COMM_WORLD);
        MPI_Allreduce(&mm.max_val, &global_max, 1, MPI_INT, MPI_MAX, MPI_COMM_WORLD);

        long long max_mag = max(abs(global_min), abs(global_max));
        for (long long exp = 1; max_mag / exp > 0 && exp > 0; exp *= 10) {
            counting_sort_neg(local_data, exp);
        }
    }

    // Gather sorted chunks
    vector<int> recvcounts, gather_displs, gathered_data;
    if (rank == 0) {
        recvcounts.resize(size);
        gather_displs.resize(size);
    }

    MPI_Gather(&local_n, 1, MPI_INT,
        rank == 0 ? recvcounts.data() : nullptr, 1, MPI_INT,
        0, MPI_COMM_WORLD);

    if (rank == 0) {
        int total = accumulate(recvcounts.begin(), recvcounts.end(), 0);
        if (total != global_n) {
            cerr << "Mismatch in total data count during gather.\n";
            MPI_Abort(MPI_COMM_WORLD, 1);
        }
        gathered_data.resize(global_n);
        partial_sum(recvcounts.begin(), recvcounts.end() - 1, gather_displs.begin() + 1);
        gather_displs[0] = 0;
    }

    MPI_Gatherv(local_data.data(), local_n, MPI_INT,
        rank == 0 ? gathered_data.data() : nullptr,
        rank == 0 ? recvcounts.data() : nullptr,
        rank == 0 ? gather_displs.data() : nullptr,
        MPI_INT,
        0, MPI_COMM_WORLD);

    // Final merge
    if (rank == 0) {
        vector<int> sorted_output;
        k_way_merge(gathered_data, recvcounts, gather_displs, size, sorted_output);
        end_time = MPI_Wtime();
        cout << "Time : " << (end_time - start_time) << " seconds." << endl;
        auto print_segment = [](const vector<int>& v, bool tail = false) {
            int start = tail ? max(0, (int)v.size() - 20) : 0;
            int end = tail ? v.size() : min(20, (int)v.size());
            for (int i = start; i < end; ++i) cout << v[i] << " ";
            cout << endl;
            };
        cout << "First 20 elements: "; print_segment(sorted_output);
        cout << "Last 20 elements: "; print_segment(sorted_output, true);
    }
}




```


```c
#ifndef RADIX_SORT_H
#define RADIX_SORT_H

#include <vector>  
#include <string> 

// Forward declaration of the struct (definition likely in .cpp)
struct MinMax;

// Use std::vector in all declarations
MinMax get_min_max(const std::vector<int>& arr);

void counting_sort_neg(std::vector<int>& arr, long long exp);

void k_way_merge(const std::vector<int>& data,
    const std::vector<int>& counts, // MPI typically uses int counts/displs
    const std::vector<int>& displs,
    int k,
    std::vector<int>& sorted_output);

// Consider renaming radix_sort() to something more specific if needed,
// like parallel_radix_sort_manual_merge() to match the .cpp function name.
// Sticking with radix_sort() as per the original header for now.
void radix_sort();


#endif // RADIX_SORT_H
```