#include <cstdio>
#include <limits>
#include <cstring>
#include <cstdlib>
#include <unistd.h>
#include <iostream>
#include <string.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <sys/wait.h>

#define SIZE 10

using namespace std;

void* sharedMem(int shmId);
int makeSemSet(key_t key);
void delSemSet(int semId);

const int ServerSemId = 0;
const int ClientSemId = 1;
const int KillSemId = 2;
const int ClientErrorSemId = 3;

int main()
{
    pid_t pid;
    void *memId;
    int semId, shmId;
    key_t semKey, shmKey;
    struct sembuf semSet;
    struct shmid_ds shmStruct;
    const char path[] = "/home";
    const int keySemId = 1;
    const int keyShmId = 2;

    // ftok - преобразовывает имя файла и идентификатор проекта
    // в ключ для системных вызовов
    semKey = ftok(path, keySemId);
    if (semKey == (key_t)-1)
    {
        cerr << "Key semaphore id is not created" << endl;
        exit(EXIT_FAILURE);
    }

    shmKey = ftok(path, keyShmId);
    if (shmKey == (key_t)-1)
    {
        cerr << "Key share memory id is not created" << endl;
        exit(EXIT_FAILURE);
    }

    semId = makeSemSet(semKey);
    if (semId == -1)
    {
        cerr << "Semaphore set is not created" << endl;
        exit(EXIT_FAILURE);
    }

    shmId = shmget(shmKey, SIZE, IPC_CREAT | SHM_R | SHM_W);
    if (shmId == -1)
    {
        cerr << "Share memory segment id is not created" << endl;
        delSemSet(semId);
        exit(EXIT_FAILURE);
    }

    memId = (char*)sharedMem(shmId);
    if (memId == NULL)
    {
        cerr << "Memory is not allocated" << endl;
        delSemSet(semId);
        shmctl(shmId, IPC_RMID, &shmStruct); // IPC_RMID - пометка сегмента как удаленого
        exit(EXIT_FAILURE);
    }

    pid = fork();
    switch(pid)
    {
    case -1:
    {
        cerr << "Error of creating semaphore set" << endl;
        delSemSet(semId);
        shmdt(memId); // освобождаем память
        shmctl(shmId, IPC_RMID, &shmStruct);
        exit(EXIT_FAILURE);
    }
    case 0: // клиент
    {
        while(true)
        {
            // ждем пока сервер получит входные данные
            semSet.sem_num = ServerSemId;
            semSet.sem_op  = -1; // уменьшить значение семафора на абсолютную величину
            semSet.sem_flg = SEM_UNDO; // обратная операция при закрытии процесса
            semop(semId, &semSet, 1);

            memId = (char*)sharedMem(shmId);
            if (memId == NULL)
            {
                cerr << "Memory is not allocated" << endl;
                delSemSet(semId);
                shmctl(shmId, IPC_RMID, &shmStruct); // IPC_RMID - пометка сегмента как удаленого
                exit(EXIT_FAILURE);
            }

            cout << "Client received: " << (char*)memId << endl;

            // уведомить, если клиент считал память
            semSet.sem_num = ClientSemId;
            semSet.sem_op  = 1;
            semSet.sem_flg = SEM_UNDO;
            semop(semId, &semSet, 1);
        }
        exit(EXIT_SUCCESS);
    }
    default: // сервер
    {
        int position = 0;
        bool input = true;
        string bufferString;
        bufferString.resize(SIZE, '\0');

        while(true)
        {
            memset(memId, '\0', 1);
            if(input)
            {
                position = 0;
                cout << "Server: Enter string" << endl;
                getline(cin, bufferString);
                input = false;
            }

            string tempBuffer;
            int len = 0;
            tempBuffer.append(bufferString, 0, SIZE - 1); // добавляет символы в конец строки
            position = tempBuffer.length();
            strcpy((char*)memId, const_cast<char*>(tempBuffer.c_str()));

            tempBuffer.clear();
            len = bufferString.length() - position;
            if (len > 0)
            {
                tempBuffer.append(bufferString, position, len);
            }
            bufferString = tempBuffer;

            // уведомить о том, что сервер принял входные данные
            semSet.sem_num = ServerSemId;
            semSet.sem_op  = 1;
            semSet.sem_flg = SEM_UNDO;
            semop(semId, &semSet, 1);

            // подождать пока клиент прочитает
            semSet.sem_num = ClientSemId;
            semSet.sem_op  = -1;
            semSet.sem_flg = SEM_UNDO;
            semop(semId, &semSet, 1);

            // проверка на наличии ошибки у клиена
            if(semctl(semId, ClientErrorSemId, GETVAL) > 0)
            {
                cout << "Error" << endl;
                break;
            }

            if(bufferString.empty())
            {
                input = true;
                cout << "Tap '0' to exit" << endl;
                cout << "Tap another symbol to continue" << endl;

                if(cin.get() == '0')
                {
                    semSet.sem_num = KillSemId;
                    semSet.sem_op  = 1;
                    semSet.sem_flg = SEM_UNDO;
                    semop(semId, &semSet, 1);
                    waitpid(pid, NULL, 0);
                    delSemSet(semId);
                    shmdt(memId); // отстыковывает сегмент разделяемой памяти по адресу
                    shmctl(shmId, IPC_RMID, &shmStruct);
                    return 0;
                }
                bufferString.clear();
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
                cout << endl;
            }
        }
    }
    }
}

void *sharedMem(int shmId)
{
    void *memId;
    memId = shmat(shmId, NULL, 0); // shmat - работа с разделяемой памятью
    return memId;
}

void delSemSet(int semId)
{
    semctl(semId, 0, IPC_RMID, NULL); // semctl - производит операции управления семафорами
}

int makeSemSet(key_t key)
{
    int semId;
    int setSemGet = 0;
    int setArray[1] = {0}; // !!!! SEM_QUANTITY = 1
    semId = semget(key, 1, IPC_CREAT | SHM_R | SHM_W);  // semget - считывает идентификатор набора семафоров
    if (semId != -1)
    {
        setSemGet = semctl(semId, 0, SETALL, setArray); // semctl - производит операции управления семафорами
        if(setSemGet == -1)
            return setSemGet;
        else
            return semId;
    }
    else
    {
        cout << "Semaphore set is not created..." << endl;
        return -1;
    }
}
