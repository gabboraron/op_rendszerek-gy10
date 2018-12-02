# GY10

## Osztott memória
> mappa: `gy6>szemafor.c`

>Ha több folyamat kommunikál akkor szinkronizálni kell az aszinkronnál is. Az osztott memória a leggyorsabbb két vég közötti kommunikációra, de egyes mikrokontrollerek nem támogatják, ezért nem elég csak azt ismerni.

> Ahhoz hogy az osztott memórián keresztül tudjunk biztonságosan adatot közölni zárni kell az olvasás előtt, nehogy a másik fél beleírjon valami értéket, ezért védjük szemaforral.
> De szemaforral más is védhető amikor szinkronizációra van szükség, pl: fájlba írás - olvasás ha a második folyamat olyant olvasna ami még nincs beírva.

### Kulcs generálás
````C
kulcs=ftok(argv[0],1); //ftok(egy létező fájl az [argv0 - sajátmaga], 1..16ig számok <- max 16 értéke lehet);
sh_mem_id=shmget(kulcs,MEMSIZE,IPC_CREAT|S_IRUSR|S_IWUSR); // shmget(kulcs, memóriaméret, jogosultság);
````
Az osztott memóriához a kulcs egyedi az oprendszer biztosítja, aki ismeri ak ulcsot az hozzáfér az osztott memóriához.

````C
s = shmat(sh_mem_id,NULL,0); // az osztott memóriához próbálunk hozzzáférni az s változón keresztül, az s változón keresztül adja vissza.
````
létrehozható: `int*x` ami indexelhető tömb lehet.

### Szemafor
A kritikus szakasz lezárására, és a másik folyamat megkérdi, hogy beléphet-e és amíg az eredi folyamt fel nem engedi addig várakozik.
````C
semid = szemafor_letrehozas(argv[0],0); // sem state is down!!!
````
Milyen állapotban jön létre a szemafor?
Ha nyitottként jön létre akkor a következő folyamat le tudja zárni, csak a következő lehet író és olvasó is! Az kell zárja az osztott memóriát aki írni akar bele! HA végzett akkor kell a másiknak írni.
`szemafor_letrehozas(argv[0],0)` jelen esetben zárva jön létre, a második `0` biztosítja.


````C
if(child>0){            //szülő
       char buffer[] = "I like Illes (Pop group:)!\n";
    printf("Szulo indul, kozos memoriaba irja: %s\n",buffer);
       sleep(4);       // child waits during sleep | zhban ne!
       strcpy(s,buffer); //szöveget ír az osztott memóriába
    printf("Szulo, szemafor up!\n");
       szemafor_muvelet(semid,1); // Up
       shmdt(s);    // release shared memory
       wait(NULL);       
       szemafor_torles(semid);
    shmctl(sh_mem_id,IPC_RMID,NULL);
    } else if ( child == 0 ) {    //gyerek
    
       // critical section
    printf("Gyerek: Indula szemafor down!\n");
       szemafor_muvelet(semid,-1); // down, wait if necessary
       printf("Gyerek, down rendben, eredmeny: %s",s);  
       szemafor_muvelet(semid,1); // szemafor up      hogy más se férjen hozzá..
       // end of critical section  
       shmdt(s);
    }
````

#### Szemafor létrehozása:
````C
int szemafor_letrehozas(const char* pathname,int szemafor_ertek){
    int semid;
    key_t kulcs; ezzel a kulccsal több is létrehozható!
    
    kulcs=ftok(pathname,1);    

//szemafor család létrehozása
    if((semid=semget(kulcs,1,IPC_CREAT|S_IRUSR|S_IWUSR ))<0) //semget(kulcs, szemaforok száma, jogosultság) <- ezzel hozható létre szemfor, nem állítható a kezdőértéke. Mert szemaforcsalád, ezért egyesével kell beállítani a kezdőállapotot
    perror("semget");
    // semget 2. parameter is the number of semaphores   
    if(semctl(semid,0,SETVAL,szemafor_ertek)<0)    //0= first semaphores //szemafor értékének beállítása: semctl(szemafor id,0,SETVAL,szemafor erteke)
        perror("semctl");
       
    return semid;
}
`````
#### Szemafor műveletek

````C
void szemafor_muvelet(int semid, int op){
    struct sembuf muvelet;
    
    muvelet.sem_num = 0; //hanyas számú szemafort állítjuk
    muvelet.sem_op  = op; // op=1 up, op=-1 down  <-szemafor beállítása
    muvelet.sem_flg = 0; //normál esetben: 0 ez biztosítja hogy várakozni fog, nem lezárható addig amíg nem kerül nyitott állapotba
    
    if(semop(semid,&muvelet,1)<0) // 1 number of sem. operations
    perror("semop");        
}
````

### Fontosabb műveletek:
Szemafor létrehozása: `semget(kulcs, szemaforok száma, jogosultság)`
Szemafor beállítása: `semctl(semid,0,SETVAL,szemafor_ertek)`
`ipcs` - linux parancs, kiírja ha valami gond van a törléssel, kiírja ha bennmaradt osztott memória, szemafor, üzenetsor.
`ipcrm` - ezel törölhető az egyes ipcsvel meghatározott dolgokat. pl: `ipcrm -s szemafor_id`





