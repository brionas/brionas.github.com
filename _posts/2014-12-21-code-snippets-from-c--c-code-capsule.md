---
layout: post
title: "Code snippets from C & C++ Code Capsule"
description: ""
category: 
tags: []
refer_author: Qun-hu Sun
---
Recently I had read the book [C & C++ Code Capsules](http://www.amazon.com/Code-Capsules-Guide-Practitioners/dp/0135917859) by Chuck Allison, there are codes related with Class pointer, Exception handling, Assert etc. We can learn a lot from it and use them in our project. Below are some code snippets from it.

1. Pointer to member function

        #include<iostream>  
        using namespace std;  
        class C  
        {  
            public:  
            void f(){ cout<<"C::f\n";}  
            void g(){cout<<"C::g\n";}  
        };  
           
        int main()  
        {  
            C c;  
            void (C::*pmf)()=&C::f;  
            (c.*pmf)();  
            pmf=&C::g;  
            (c.*pmf)();  
        }

    <span/>

        #include<iostream> 
        using namespace std; 
           
        class Object{ 
        public: 
            void retrieve(){ 
                cout<<"Object::retrieve"<<endl; 
            } 
            void insert(){ 
                cout<<"Object::insert"<<endl; 
            } 
            void update(){ 
                cout<<"Object::update"<<endl; 
            } 
            void process(int choice); 
           
        private: 
            typedef void (Object::*Omf)(); 
            static Omf farray[3]; 
        }; 
           
        Object::Omf Object::farray[3]={&Object::retrieve,&Object::insert,&Object::update}; 
           
        void Object::process(int choice){ 
            if(0<=choice && choice<=2) 
                (this->*farray[choice])(); 
        } 
           
        int show_menu() 
        { 
            cout<<"1. retrieve\n"; 
            cout<<"2. insert\n"; 
            cout<<"3. update\n"; 
            cout<<"4. quit\n"; 
            cout<<"Please input a num\n"; 
            int n; 
            cin>>n; 
            return n; 
        } 
           
        int main(){ 
            int show_menu();  
               
            Object o; 
              for(;;){ 
                int choice=show_menu(); 
                if(1<=choice && choice<=3) 
                    o.process(choice-1); 
                else if(choice==4) 
                    break; 
            } 
            return 0; 
        }

2. Implement your own ASSERT macro
        // assert.h 
        #ifndef ASSERT_H 
        #define ASSERT_H 
        
        
        extern void _assert(char*, char*, long); 
        #define assert(cond) \ 
        ((cond) \ 
            ? (void) 0 \ 
            : _assert(#cond, __FILE__, __LINE__)) 
        #endif 
        
        // assert.cpp
        #include <stdio.h> 
        #include <stdlib.h> 
        #include "assert.h" 
        
        
        void _assert(char* cond, char* fname, long lineno) 
        {
            fprintf(stderr, "Assertion failed: %s, file %s, line %ld\n", cond, fname, lineno); 
            abort(); 
        }
        
        int main() 
        { 
            assert(3>4); 
            return 0; 
           
        }  
3. Use setjmp to handle exception

        #include <stdio.h> 
        #include <setjmp.h> 
           
        jmp_buf jumper; 
        void exception(); 
        int  deal_exception(); 
           
        main() 
        { 
             int value = 0; 
             int i = 0; 
           
             value = setjmp(jumper); 
           
             if ( 0 == value ) { 
                 exception(); 
             } 
             else { 
                 switch ( value ) 
                 { 
                    case 1: 
                        printf( "solve exception[%d]\n",value ); 
                        break; 
                    case 2: 
                        printf( "solve exception[%d]\n",value ); 
                        break; 
                    case 3: 
                        printf( "solve exception[%d]\n",value ); 
                        break; 
                   default: 
                        printf( "solve exception[%d]\n",value ); 
                        break; 
                } 
            } 
        } 
        
        void exception() 
        { 
            int _err_no = 3; 
           
            if ( _err_no == 3 ) { 
                printf("exception[%d]\n",_err_no); 
                longjmp(jumper,_err_no); 
            } 
           
            return; 
        }
        
4. Template specialization

        ///////////////////////////////////////////////////////////////////////////
        /* Function template specialization */
        #include <iostream> 
        #include <string.h> 
        #include <string> 
           
        using namespace std; 
           
        template<class T> 
        size_t bytes( T& t) 
        { 
            cout<<"(using primary template)\n"; 
            return sizeof t; 
        } 
           
        size_t bytes( char*& t) 
        { 
            cout<<"(using char* overload)\n"; 
            return sizeof t; 
        } 
           
        template<> 
        size_t bytes<wchar_t*>( wchar_t*& t) 
        { 
            cout<<"(using wchar_t* specialization)\n"; 
            return sizeof 2*(wcslen(t)+1); 
        } 
           
        template<> 
        size_t bytes<>( string& s) 
        { 
            cout<<"(using string explicit specialization)\n"; 
            return sizeof s; 
        } 
           
        template<> 
        size_t bytes<float>( float& s) 
        { 
            cout<<"(using float explicit specialization)\n"; 
            return sizeof s; 
        } 
               
        int main() 
        { 
            int i; 
            cout<<"bytes in i:"<<bytes(i)<<endl; 
            const char* s="Hello"; 
            cout<<"bytes in s:"<<bytes(s)<<endl; 
            const wchar_t* w = (wchar_t*)"goodbye"; 
            cout<<"bytes in w:"<<bytes(w)<<endl; 
            string t; 
            cout<<"bytes in t:"<<bytes(t)<<endl; 
           
            float f; 
            cout<<"bytes in f:"<<bytes(f)<<endl; 
           
            double d; 
            cout<<"bytes in d:"<<bytes(d)<<endl; 
           
            return 0; 
           
        }
         
        ///////////////////////////////////////////////////////////////////////////
        /* Class template specialization */
        #include <iostream> 
        using namespace std; 
           
        template<class T, class U> 
        class A 
        { 
        public: 
            A(){cout<<"Primary template\n";} 
        }; 
           
        template<class T, class U> 
        class A<T*, U> 
        { 
        public: 
            A(){cout <<"<T*,U> partial specialization\n";} 
        }; 
           
        template<class T> 
        class A<T, T> 
        { 
        public: 
            A(){cout <<"<T,T> partial specialization\n";} 
        }; 
           
        template<class U> 
        class A<int, U> 
        { 
        public: 
            A(){cout <<"<int,U> partial specialization\n";} 
        }; 
           
        int main() 
        { 
            A<char,int> a1; 
            A<char*,int> a2; 
            A<float,float> a3; 
            A<int, float> a4; 
           
            return 0; 
        }
5. Change member variable in const member function

        // Method 1: const cast this pointer
        // Method 2: Use mutable
        #include <iostream> 
        using namespace std; 
        class B 
        { 
        public: 
            B() {state =0;pos=0;} 
            void f() const; 
            void p() {cout<<"state= "<<state<<" pos= "<<pos<<endl;} 
        private: 
            int state; 
            mutable int pos; 
        }; 
        void B::f() const 
        { 
            // ((B*)this)->state=1; 
            const_cast<B*>(this)->state=1; 
            pos = 1; 
        } 
           
        int main() 
        { 
            B b; 
            b.p(); 
            b.f(); 
            b.p(); 
           
            return 0; 
        }
        
6. Signal Handle

        #include <stdio.h> 
        #include <signal.h> 
        void ctrlc_handler(int sig); 
           
        volatile sig_atomic_t ccount = 0; 
           
        int main() 
        { 
            char buf[100]; 
            if(signal(SIGINT, ctrlc_handler) == SIG_ERR) 
            { 
                fputs("Error installing ctrlc handler\n", stderr); 
            } 
           
            while(gets(buf)) 
            { 
                puts(buf); 
            } 
           
            signal(SIGINT, SIG_DFL); 
            printf("You pressed ctrlc %d times\n", ccount); 
           
            return 0; 
        } 
           
        void ctrlc_handler(int sig) 
        { 
            signal(sig, ctrlc_handler); 
           
            ++ccount; 
            return; 
        }
7. Exception handle use Resource Acquisition Is Initialization (RAII)

        /*
        exception:
            logic_error
                domain_error
                invalid_argument
                length_error
                out_of_range
            runtime_error
                range_error
                overflow_error
                underflow_error
            bad_alloc
            bad_cast
            bad_exception
            bad_typeid
        */ 
        #include <cstdio> 
        #include <memory> 
        using namespace std; 
        class File 
        { 
            FILE* f; 
        public: 
            File(const char* fname, const char* mode) 
            { 
                f = fopen(fname,mode); 
                if(!f) 
                    throw 1; 
            } 
            ~File() 
            { 
                if(f) 
                { 
                    fclose(f); 
                    puts("File closed"); 
                } 
            } 
        }; 
           
           
        int main() 
        { 
            void f(const char*); 
            try 
            { 
                //f("file0.txt"); 
                f("file1.txt"); 
            } 
            catch(int x) 
            { 
                printf("Caught exception:%d\n", x); 
            } 
           
            return 0; 
        } 
           
           
        void f(const char* fname) 
        { 
            auto_ptr<File> xp(new File(fname,"r")); 
            puts("Processing file..."); 
            throw 2; 
        }

8. None standard containers
        
        #include <valarray> 
        #include <iostream> 
        #include <algorithm>  
           
        using namespace std; 
           
        void print(double i) 
        { 
            cout << i <<" "; 
        } 
        int main() 
        { 
            const int N = 10; 
            const double values[N] = {0,1,2,3,4,5,6,7,8,9}; 
            const valarray<double> v1(values,N);
           
            for_each(&v1[0], &v1[10], ptr_fun(print)); 
            cout<<endl; 
           
            cout << "min: "<<v1.min()<<endl; 
            cout << "max: "<<v1.max()<<endl; 
            cout << "sum: "<<v1.sum()<<endl; 
           
               
           
            return 0; 
           
        }
9. Handle Memory Allocation exception

        // Method 1: try{} catch(const bad_alloc& x)
         
        // Method 2: Use set_new_handler
        #include <iostream>  
        #include <stdlib.h>  
        using namespace std;  
           
        inline void my_handler()  
        {  
            cout << "Memory exhausted" <<endl;  
            abort();  
        }  
           
        int main()  
        {  
            set_new_handler(my_handler);  
           
            for(int i=0;;++i)  
            {  
                (void) new double[100];  
                if((i+1) % 10 == 0)  
                    cout << (i+1) << " allocations" <<endl;  
            }  
           
            return 0;  
        }
10. Never run 9 on your computer!
