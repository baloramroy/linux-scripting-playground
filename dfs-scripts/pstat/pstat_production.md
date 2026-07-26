```bash
#!/bin/bash
rm -rf ps.tmp
touch  ps.tmp

#after this add line to  get component name

#echo kpp-apigw-INST_1               >>ps.tmp
#echo kpp-apigw-INST_2               >>ps.tmp
#echo kpp-apigw-INST_3               >>ps.tmp
echo kpp-cms                 >>ps.tmp
echo kpp-cp                  >>ps.tmp
echo kpp-cs                  >>ps.tmp
echo kpp-davs                >>ps.tmp
echo kpp-dfs                 >>ps.tmp
echo kpp-drs                 >>ps.tmp
echo kpp-dmscore             >>ps.tmp
echo kpp-extch       	     >>ps.tmp
echo kpp-extchSMS            >>ps.tmp
echo kpp-ias                 >>ps.tmp
echo kpp-kms                 >>ps.tmp
echo kpp-knotify             >>ps.tmp
echo kpp-kod                 >>ps.tmp
echo kpp-map                 >>ps.tmp
echo kpp-pcs                 >>ps.tmp
echo kpp-spg                 >>ps.tmp
echo kpp-tms                 >>ps.tmp
echo kpp-tsp                 >>ps.tmp
echo kpp-bds		     >>ps.tmp
echo kpp-ecs		     >>ps.tmp
echo KPP-rms		     >>ps.tmp
echo KPP-rpg		     >>ps.tmp
echo KPP-mps		     >>ps.tmp
echo KPP-bkofc		     >>ps.tmp
echo KPP-utilityservice	     >>ps.tmp
#echo kpp-apigwext	     >>ps.tmp
#echo kpp-npsb_recon	     >>ps.tmp

printf "\033[32m================================================================================\033[0m\n"              >> pstat.tmp
printf "\033[32mProcess Status Monitoring Script. [ KONA SOFTWARE LAB LTD.]            \033[0m\n"              >> pstat.tmp
printf "\033[32m================================================================================\033[0m\n"              >> pstat.tmp
printf "\033[32m%-16s%8s %8s %8s %2s %8s %8s %10s \033[0m\n" "PROCESS" "UID" "PID" "PPID" "C" "STIME" "TTY" "TIME" >> pstat.tmp
printf "\033[32m================================================================================\033[0m\n"              >> pstat.tmp

found=0
total=0

while read line
do	
    for LINE in $line
        do
                total=`expr $total + 1`
                \rm -rf awk_$$.tmp
                #ps -ef | grep -w $line | awk '{ if ($8!="grep" && $8!="tail") printf "%8s %5d %5d %2d %8s %8s %10s\n", $1, $2, $3, $4, $5, $6, $7 }' >  awk.tmp
                #ps -ef | grep -w $line | awk '{ if ($3 == 1) printf "%8s %5d %5d %2d %8s %8s %10s\n", $1, $2, $3, $4, $5, $6, $7 }' >  awk.tmp
                #ps -ef | grep -w $line | grep -v su | awk '{ if ($8!="grep" && $8!="tail") printf "%8s %8d %8d %2d %8s %8s %10s\n", $1, $2, $3, $4, $5, $6, $7 }' >  awk_$$.tmp
                #ps -ef | grep -iw $line | awk '{ if ($8!="grep" && $8!="tail") printf "%8s %8d %8d %2d %8s %8s %10s\n", $1, $2, $3, $4, $5, $6, $7 }' >  awk_$$.tmp
		if [ ! -s awk_$$.tmp ]
                then
                        printf "\033[31m%-16s\033[0m" $line >> pstat.tmp
			printf "\033[31m%8s %5s %5s %2s %8s %8s %10s\033[0m\n" "-" "-" "-" "-" "-" "-" "-" > awk_$$.tmp
                        cat awk_$$.tmp >> pstat.tmp
                        found=`expr $found + 1`
                else
                        printf "\033[31m%-16s\033[0m" $line >> pstat.tmp
                        sed '2,$s/^/                /' awk_$$.tmp >> pstat.tmp
                fi
    done
done < ps.tmp

let "alive = total - found"

if [ -e pstat.tmp ]
then
        printf "\033[32m================================================================================\033[0m\n" >> pstat.tmp
        printf "\033[33mProcess Total = %d , Alive = %d , Down = %d \033[0m\n" "$total" "$alive" "$found">> pstat.tmp
        printf "\033[32m================================================================================\033[0m\n" >> pstat.tmp
        cat pstat.tmp
else
        echo No Manager process !!!
fi

\rm -rf ps.tmp
\rm -rf awk_$$.tmp
\rm -rf pstat.tmp

```