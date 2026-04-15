#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <semaphore.h>
#include <time.h>
#include <unistd.h>

#define MAX_STR_LEN 64
#define PRODUCE_COUNT 50
#define SHARED_BUF_SIZE 8
#define PROCESSED_BUF_SIZE 16
#define WORKER_COUNT 3

typedef struct { char data[MAX_STR_LEN]; } item_t;

typedef struct {
    item_t buf[SHARED_BUF_SIZE];
    int head, tail, count;
    sem_t empty, full;
    pthread_mutex_t mtx;
} shared_buffer_t;

typedef struct {
    item_t buf[PROCESSED_BUF_SIZE];
    int head, tail, count;
    sem_t empty, full;
    pthread_mutex_t mtx;
} processed_buffer_t;

shared_buffer_t shared;
processed_buffer_t processed;
int producer_done = 0;
int produced_total = 0, consumed_total = 0;
pthread_mutex_t produced_mtx = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t consumed_mtx = PTHREAD_MUTEX_INITIALIZER;
static const char charset[] = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";

void random_string(char *s, size_t maxlen){
    int len = (rand() % (maxlen - 1)) + 1;
    for(int i=0;i<len;i++) s[i]=charset[rand()%(sizeof(charset)-1)];
    s[len]='\0';
}

void shared_init(shared_buffer_t *b){
    b->head=b->tail=b->count=0;
    sem_init(&b->empty,0,SHARED_BUF_SIZE);
    sem_init(&b->full,0,0);
    pthread_mutex_init(&b->mtx,NULL);
}

void processed_init(processed_buffer_t *b){
    b->head=b->tail=b->count=0;
    sem_init(&b->empty,0,PROCESSED_BUF_SIZE);
    sem_init(&b->full,0,0);
    pthread_mutex_init(&b->mtx,NULL);
}

void shared_put(shared_buffer_t *b,const item_t *it){
    sem_wait(&b->empty);
    pthread_mutex_lock(&b->mtx);
    b->buf[b->tail]=*it;
    b->tail=(b->tail+1)%SHARED_BUF_SIZE;
    b->count++;
    pthread_mutex_unlock(&b->mtx);
    sem_post(&b->full);
}

int shared_get(shared_buffer_t *b,item_t *it){
    while(1){
        if(sem_trywait(&b->full)==0){
            pthread_mutex_lock(&b->mtx);
            *it = b->buf[b->head];
            b->head=(b->head+1)%SHARED_BUF_SIZE;
            b->count--;
            pthread_mutex_unlock(&b->mtx);
            sem_post(&b->empty);
            return 0;
        } else {
            pthread_mutex_lock(&produced_mtx);
            int done = producer_done && (b->count==0);
            pthread_mutex_unlock(&produced_mtx);
            if(done) return -1;
            usleep(1000);
        }
    }
}

void processed_put(processed_buffer_t *b,const item_t *it){
    sem_wait(&b->empty);
    pthread_mutex_lock(&b->mtx);
    b->buf[b->tail]=*it;
    b->tail=(b->tail+1)%PROCESSED_BUF_SIZE;
    b->count++;
    pthread_mutex_unlock(&b->mtx);
    sem_post(&b->full);
}

int processed_get(processed_buffer_t *b,item_t *it){
    if(sem_trywait(&b->full)==0){
        pthread_mutex_lock(&b->mtx);
        *it = b->buf[b->head];
        b->head=(b->head+1)%PROCESSED_BUF_SIZE;
        b->count--;
        pthread_mutex_unlock(&b->mtx);
        sem_post(&b->empty);
        return 0;
    }
    return -1;
}

void reverse_str(const char *in,char *out){
    size_t n=strlen(in);
    for(size_t i=0;i<n;i++) out[i]=in[n-1-i];
    out[n]='\0';
}

int is_palindrome(const char *s){
    size_t i=0,j=strlen(s);
    if(j==0) return 1;
    j--;
    while(i<j){ if(s[i]!=s[j]) return 0; i++; j--; }
    return 1;
}

void char_freq(const char *s,int freq[256]){
    for(int i=0;i<256;i++) freq[i]=0;
    for(const unsigned char *p=(const unsigned char*)s; *p; p++) freq[*p]++;
}

void format_processed(const char *orig,char *out,size_t outlen){
    char rev[MAX_STR_LEN];
    reverse_str(orig,rev);
    int pal=is_palindrome(orig);
    int freq[256];
    char_freq(orig,freq);
    char freqpart[512]; freqpart[0]='\0';
    int first=1;
    for(int c=32;c<127;c++){
        if(freq[c]>0){
            if(!first) strncat(freqpart,",",sizeof(freqpart)-strlen(freqpart)-1);
            char tmp[64]; snprintf(tmp,sizeof(tmp),"%c:%d",(char)c,freq[c]);
            strncat(freqpart,tmp,sizeof(freqpart)-strlen(freqpart)-1);
            first=0;
        }
    }
    snprintf(out,outlen,"orig=%s;rev=%s;pal=%d;freq=%s",orig,rev,pal,freqpart);
}

void *producer_thread(void *arg){
    (void)arg;
    for(int i=0;i<PRODUCE_COUNT;i++){
        item_t it;
        random_string(it.data,MAX_STR_LEN);
        shared_put(&shared,&it);
        pthread_mutex_lock(&produced_mtx);
        produced_total++;
        pthread_mutex_unlock(&produced_mtx);
        usleep((rand()%100)*1000);
    }
    pthread_mutex_lock(&produced_mtx);
    producer_done=1;
    pthread_mutex_unlock(&produced_mtx);
    return NULL;
}

void *worker_thread(void *arg){
    (void)arg;
    while(1){
        item_t it;
        if(shared_get(&shared,&it)!=0) break;
        item_t out;
        format_processed(it.data,out.data,sizeof(out.data));
        processed_put(&processed,&out);
        pthread_mutex_lock(&consumed_mtx);
        consumed_total++;
        pthread_mutex_unlock(&consumed_mtx);
        usleep((rand()%200)*1000);
    }
    return NULL;
}

void *printer_thread(void *arg){
    (void)arg;
    while(1){
        item_t it;
        if(processed_get(&processed,&it)==0){
            printf("%s\n",it.data);
        } else {
            pthread_mutex_lock(&produced_mtx);
            int done = producer_done && (consumed_total==produced_total);
            pthread_mutex_unlock(&produced_mtx);
            if(done && processed.count==0) break;
            usleep(1000);
        }
    }
    return NULL;
}

int main(void){
    srand((unsigned int)time(NULL));
    shared_init(&shared);
    processed_init(&processed);
    pthread_t prod;
    pthread_t workers[WORKER_COUNT];
    pthread_t printer;
    pthread_create(&prod,NULL,producer_thread,NULL);
    for(int i=0;i<WORKER_COUNT;i++) pthread_create(&workers[i],NULL,worker_thread,NULL);
    pthread_create(&printer,NULL,printer_thread,NULL);
    pthread_join(prod,NULL);
    for(int i=0;i<WORKER_COUNT;i++) pthread_join(workers[i],NULL);
    pthread_join(printer,NULL);
    printf("Produced: %d, Processed: %d\n",produced_total,consumed_total);
    return 0;
}
