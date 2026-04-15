#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

int main(void) {
    int n, i, thread_count;
    double *a, *b;
    double dot = 0.0;

    printf("Enter vector length (n): ");
    if (scanf("%d", &n) != 1 || n <= 0) return 1;
    printf("Enter number of threads: ");
    if (scanf("%d", &thread_count) != 1 || thread_count <= 0) return 1;

    a = malloc(sizeof(double) * n);
    b = malloc(sizeof(double) * n);
    if (!a || !b) { perror("malloc"); return 1; }

    /* Initialize vectors (example values) */
    for (i = 0; i < n; ++i) {
        a[i] = i + 1.0;      /* 1.0, 2.0, ... */
        b[i] = 1.0 / (i + 1.0); /* 1.0, 1/2, 1/3, ... */
    }

    omp_set_num_threads(thread_count);
    #pragma omp parallel for reduction(+:dot)
    for (i = 0; i < n; ++i) {
        dot += a[i] * b[i];
    }

    printf("Dot product = %f\n", dot);

    free(a);
    free(b);
    return 0;
}
