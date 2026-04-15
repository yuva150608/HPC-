#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <omp.h>

#define LINE_MAX 1024
#define QUEUE_SIZE 64
#define MAX_FILES 32

/* Shared circular queue */
char queue[QUEUE_SIZE][LINE_MAX];
int q_head = 0;   /* next remove index */
int q_tail = 0;   /* next insert index */
int q_count = 0;  /* items in queue */

omp_lock_t qlock;
int finished_producers = 0;
int num_producers = 0;
char *filenames[MAX_FILES];
char search_word[256];

void enqueue(const char *line) {
    while (1) {
        omp_set_lock(&qlock);
        if (q_count < QUEUE_SIZE) {
            strncpy(queue[q_tail], line, LINE_MAX-1);
            queue[q_tail][LINE_MAX-1] = '\0';
            q_tail = (q_tail + 1) % QUEUE_SIZE;
            q_count++;
            omp_unset_lock(&qlock);
            return;
        }
        omp_unset_lock(&qlock);
        /* queue full -> yield briefly */
        #pragma omp flush
    }
}

int dequeue(char *out_line) {
    /* returns 1 if an item was removed, 0 if queue empty (but producers may still run) */
    omp_set_lock(&qlock);
    if (q_count > 0) {
        strncpy(out_line, queue[q_head], LINE_MAX-1);
        out_line[LINE_MAX-1] = '\0';
        q_head = (q_head + 1) % QUEUE_SIZE;
        q_count--;
        omp_unset_lock(&qlock);
        return 1;
    }
    omp_unset_lock(&qlock);
    return 0;
}

void producer(int id) {
    FILE *fp = fopen(filenames[id], "r");
    if (!fp) {
        #pragma omp critical
        fprintf(stderr, "Producer %d: cannot open %s\n", id, filenames[id]);
        #pragma omp atomic
        finished_producers++;
        return;
    }
    char line[LINE_MAX];
    while (fgets(line, LINE_MAX, fp)) {
        enqueue(line);
        #pragma omp critical
        printf("Producer %d enqueued: %s", id, line);
    }
    fclose(fp);
    #pragma omp atomic
    finished_producers++;
}

void consumer(int id) {
    char line[LINE_MAX];
    while (1) {
        /* Try to dequeue */
        if (dequeue(line)) {
            /* search for whole word or substring? use simple substring match */
            if (strstr(line, search_word) != NULL) {
                #pragma omp critical
                {
                    printf("Consumer %d match: %s", id, line);
                }
            }
        } else {
            /* queue empty: check termination */
            omp_set_lock(&qlock);
            int producers_done = (finished_producers == num_producers);
            int cnt = q_count;
            omp_unset_lock(&qlock);

            if (producers_done && cnt == 0) break; /* all done */
            /* otherwise wait briefly and retry */
            #pragma omp flush
        }
    }
}

int main(int argc, char *argv[]) {
    if (argc < 4) {
        fprintf(stderr, "Usage: %s <search_word> <num_consumers> <file1> [file2 ...]\n", argv[0]);
        return 1;
    }

    strncpy(search_word, argv[1], sizeof(search_word)-1);
    search_word[sizeof(search_word)-1] = '\0';
    int num_consumers = atoi(argv[2]);
    if (num_consumers <= 0) num_consumers = 1;

    num_producers = argc - 3;
    if (num_producers > MAX_FILES) {
        fprintf(stderr, "Too many files (max %d)\n", MAX_FILES);
        return 1;
    }

    for (int i = 0; i < num_producers; ++i) filenames[i] = argv[3 + i];

    omp_init_lock(&qlock);

    int total_threads = num_producers + num_consumers;
    #pragma omp parallel num_threads(total_threads)
    {
        int tid = omp_get_thread_num();
        if (tid < num_producers) {
            producer(tid);
        } else {
            consumer(tid - num_producers);
        }
    }

    omp_destroy_lock(&qlock);
    return 0;
}
