# Readers Writers Starve Free Solution
 
## Problem Description
 *Readers-Writers* problem is a classical problem of synchronization.In this multiple readers and writers try to access the critical section concurrently. The crititcal section can either have multiple readers or a single writer. The problem of synchronization is tackeled by using semaphores such that readers get priority and writer has to wait for all readers to complete reading. this leads to **starvation**.

## Solution 
I am employing a FIFO based solution to make a starve free *Readers-Writers* synchronization.The basic intuition is that all the processes will be waiting in a virtual queue for their chance to enter critical section. If there is a reader in critical section and another reader is waiting in queue for critical section then he is also granted access. all the consecutive readers in queue are given simultaneous access to crititcal section. If a writer is waiting in the queue he will be given chance after his predecessors in the queue complete their critical section job.

The following pseudocode presents better insight of the solution. This pseudocode uses `c++` like syntax.

## Pseudo Code

This implementation of major classes used for the solution is given below:
### Generic Queue
```cpp
template<typename T>
// This is the generic implementation of FIFO queue 
class queue{ 
    T* front=NULL;
    T* rear=NULL;
public:
    void push(int p_id){ // used to add new member to the queue
        T* p=new T(p_id);// the process is in "blocked" state when initialised
        if(rear!=NULL){
            rear->next=p;
            rear=p;  
        }else{
            front=rear=p; 
        }
    }

    int pop(){// pops the front element the queue and returns its pid
    if(front==NULL)return -1;
    int p_id=front->pid; 
    front->status=active; // This line is specific for this problem. status is the state of the process if its blocked then status is "blocked" and "active" if the process wakes up
    front=front->next;
    rear=(front==NULL)?NULL:rear;

    return p_id;
    }
};
```
A process enters this queue if it is waiting for its turn or resouces(semaphores).The processes that do not wait for other resouces(semaphores) don't enter this queue. 
when a process is inserted into this queue it is initialized with *blocked* as its default state. when popped out of the queue it becomes *active*. These are being used instead of syscalls block() and wake().

### Process_Node
This is like pcb of a process. It stores the pid, status and pointer to next process.
```cpp
enum State{active,blocked}; //This is the process state
class process_node{

    int pid;
public:
    process_node* next;
    State state;
    process_node(int p_id){
        pid=p_id;
        state=blocked;
        next=NULL;
    }
};
```

### semaphore
This class defines how wait and signal operate on semaphore.
```cpp

class semaphore{
    public:
    int instance=1;// The number of instances of a resource
    queue<process_node> q;
    semaphore(int inst){
        instance=inst;
    }

    void wait(int p_id){
        instace--;
        if(instance<0){ // if resource is not available then we block the process
            q.push(p_id); 
        }
    }

    void signal(){
        instace++;
        if(instance<=0){// we wake the process when critical section(resource) is available.
            q.pop();
        }
    }
}

```
### Global variable
These are the variables common to both readers and writeres

```cpp
int entry_count=0,exit_count=0;// These represent the number of readers entering and exiting critical section.
semaphore read_entry=new seamphore(1);// used to update entry_count mutually exclusively.
semaphore read_exit=new semaphore(1);// used to update  exit_count mutually exclusively.
semaphore mutex_write=new semaphore(0);// used to perform write mutually exclusively
bool writer=false;// This is used to indicate to readers in crititcal section that writer is waiting. the last reader leaving the critical section signals writer depending on this boolen. since we will signal only if writer is waiting.
```

### Reader
```cpp
while(true){
    int pid=new PID();// generating pid for this process

    /* **ENTRY SECTION** */
    read_entry.wait(pid);
        entry_count++; // reader entering CS.
    read_entry.signal();

    /* **** CRITICAL SECTION**** */

    read_exit.wait(pid);
        exit_count++;// reader exiting CS.
        if(waiting &&(read_entry==read_exit)){
            mutex_write.signal();//last reader signals to writer that he can enter CS if a writer is waiting.
        }
    read_exit.signal();   
    // ***REMAINDER SECTION***
}
```

### Writer

```cpp
while(true){
    int pid=new PID();// PID generation
    /* ***ENTRY SECTION*** */
    read_entry.wait(pid);//The writer is inserted into queue if he doesn't have resource access. the next processes will be after writer in queue.
    read_exit.wait(pid);//The writer waits for last reader to exit CS. 
    if(entry_count==exit_count){
        read_exit.signal();// since next processes are in queue for entry. signalling read_exit will not lead to error even if we do it befor CS.        
    }else{
        waiting=true;
        read_exit.signal();// allowing other readers to leave. no new reader enter CS.
        mutex_write.wait(pid);//waiting for readers to exit.
        read_exit.signal();
    }
    //**** CRITICAL SECTION ****
    read_entry.signal();
    // ***REMAINDER SECTION***
}
```
`CS`: CRITICAL SECTION 

when a reader process is created he is pushed to read_entry queue if CS is not available else he will continue to CS. while a reader is reading if another reader enters CS after using `read_entry` to update the *entry_count* . 
If a writer appears when the critical section is occupied he will enter `read_entry` queue so that other processes after him will proceed to CS after him. this ensures that writer does not starve.once the readers using CS leave the last reader signals writer to proceed to CS.
Similarly, if a reader process comes when a write is Using CS. the reader will enter read_entry queue. It will wait for the writer to complete and it will enter CS after him regardeless of the other process that come after the reader. 

In this no process will starve due other and hence the solution is starve free.
## References
- Operating Systems Concepts (9th Edition), Abraham Silberchatz, Peter B Galvin and Greg Gagne
- [P. J. Courtois, F. Heymans, and D. L. Parnas. 1971. Concurrent control with “readers” and “writers” Commun. ACM 14, 10 (Oct. 1971), 667–668.](https://doi.org/10.1145/362759.362813)

## Author
[Pavan Sai](https://github.com/pavansai444)
