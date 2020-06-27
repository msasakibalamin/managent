# managent
from django import forms
from django.contrib.auth.models import User
from . import models



#for admin signup
class AdminSigupForm(forms.ModelForm):
    class Meta:
        model=User
        fields=['first_name','last_name','username','password']
        widgets = {
        'password': forms.PasswordInput()
        }


#for student related form
class DoctorUserForm(forms.ModelForm):
    class Meta:
        model=User
        fields=['first_name','last_name','username','password']
        widgets = {
        'password': forms.PasswordInput()
        }
class DoctorForm(forms.ModelForm):
    class Meta:
        model=models.Doctor
        fields=['address','mobile','department','status','profile_pic']



#for teacher related form
class PatientUserForm(forms.ModelForm):
    class Meta:
        model=User
        fields=['first_name','last_name','username','password']
        widgets = {
        'password': forms.PasswordInput()
        }
class PatientForm(forms.ModelForm):
    #this is the extrafield for linking patient and their assigend doctor
    #this will show dropdown __str__ method doctor model is shown on html so override it
    #to_field_name this will fetch corresponding value  user_id present in Doctor model and return it
    assignedDoctorId=forms.ModelChoiceField(queryset=models.Doctor.objects.all().filter(status=True),empty_label="Name and Department", to_field_name="user_id")
    class Meta:
        model=models.Patient
        fields=['address','mobile','status','symptoms','profile_pic']



class AppointmentForm(forms.ModelForm):
    doctorId=forms.ModelChoiceField(queryset=models.Doctor.objects.all().filter(status=True),empty_label="Doctor Name and Department", to_field_name="user_id")
    patientId=forms.ModelChoiceField(queryset=models.Patient.objects.all().filter(status=True),empty_label="Patient Name and Symptoms", to_field_name="user_id")
    class Meta:
        model=models.Appointment
        fields=['description','status']


class PatientAppointmentForm(forms.ModelForm):
    doctorId=forms.ModelChoiceField(queryset=models.Doctor.objects.all().filter(status=True),empty_label="Doctor Name and Department", to_field_name="user_id")
    class Meta:
        model=models.Appointment
        fields=['description','status']


#for contact us page
class ContactusForm(forms.Form):
    Name = forms.CharField(max_length=30)
    Email = forms.EmailField()
    Message = forms.CharField(max_length=500,widget=forms.Textarea(attrs={'rows': 3, 'cols': 30}))



#Developed By : alamin
#facebook : fb.com/msacom.cf
#Youtube :youtube.com/lazycoders

from django.shortcuts import render,redirect,reverse
from . import forms,models
from django.db.models import Sum
from django.contrib.auth.models import Group
from django.http import HttpResponseRedirect
from django.core.mail import send_mail
from django.contrib.auth.decorators import login_required,user_passes_test
from datetime import datetime,timedelta,date


# Create your views here.
def home_view(request):
    if request.user.is_authenticated:
        return HttpResponseRedirect('afterlogin')
    return render(request,'hospital/index.html')


#for showing signup/login button for admin(by sumit)
def adminclick_view(request):
    if request.user.is_authenticated:
        return HttpResponseRedirect('afterlogin')
    return render(request,'hospital/adminclick.html')


#for showing signup/login button for doctor(by sumit)
def doctorclick_view(request):
    if request.user.is_authenticated:
        return HttpResponseRedirect('afterlogin')
    return render(request,'hospital/doctorclick.html')


#for showing signup/login button for patient(by sumit)
def patientclick_view(request):
    if request.user.is_authenticated:
        return HttpResponseRedirect('afterlogin')
    return render(request,'hospital/patientclick.html')




def admin_signup_view(request):
    form=forms.AdminSigupForm()
    if request.method=='POST':
        form=forms.AdminSigupForm(request.POST)
        if form.is_valid():
            user=form.save()
            user.set_password(user.password)
            user.save()
            my_admin_group = Group.objects.get_or_create(name='ADMIN')
            my_admin_group[0].user_set.add(user)
            return HttpResponseRedirect('adminlogin')
    return render(request,'hospital/adminsignup.html',{'form':form})




def doctor_signup_view(request):
    userForm=forms.DoctorUserForm()
    doctorForm=forms.DoctorForm()
    mydict={'userForm':userForm,'doctorForm':doctorForm}
    if request.method=='POST':
        userForm=forms.DoctorUserForm(request.POST)
        doctorForm=forms.DoctorForm(request.POST,request.FILES)
        if userForm.is_valid() and doctorForm.is_valid():
            user=userForm.save()
            user.set_password(user.password)
            user.save()
            doctor=doctorForm.save(commit=False)
            doctor.user=user
            doctor=doctor.save()
            my_doctor_group = Group.objects.get_or_create(name='DOCTOR')
            my_doctor_group[0].user_set.add(user)
        return HttpResponseRedirect('doctorlogin')
    return render(request,'hospital/doctorsignup.html',context=mydict)


def patient_signup_view(request):
    userForm=forms.PatientUserForm()
    patientForm=forms.PatientForm()
    mydict={'userForm':userForm,'patientForm':patientForm}
    if request.method=='POST':
        userForm=forms.PatientUserForm(request.POST)
        patientForm=forms.PatientForm(request.POST,request.FILES)
        if userForm.is_valid() and patientForm.is_valid():
            user=userForm.save()
            user.set_password(user.password)
            user.save()
            patient=patientForm.save(commit=False)
            patient.user=user
            patient.assignedDoctorId=request.POST.get('assignedDoctorId')
            patient=patient.save()
            my_patient_group = Group.objects.get_or_create(name='PATIENT')
            my_patient_group[0].user_set.add(user)
        return HttpResponseRedirect('patientlogin')
    return render(request,'hospital/patientsignup.html',context=mydict)






#-----------for checking user is doctor , patient or admin(by sumit)
def is_admin(user):
    return user.groups.filter(name='ADMIN').exists()
def is_doctor(user):
    return user.groups.filter(name='DOCTOR').exists()
def is_patient(user):
    return user.groups.filter(name='PATIENT').exists()


#---------AFTER ENTERING CREDENTIALS WE CHECK WHETHER USERNAME AND PASSWORD IS OF ADMIN,DOCTOR OR PATIENT
def afterlogin_view(request):
    if is_admin(request.user):
        return redirect('admin-dashboard')
    elif is_doctor(request.user):
        accountapproval=models.Doctor.objects.all().filter(user_id=request.user.id,status=True)
        if accountapproval:
            return redirect('doctor-dashboard')
        else:
            return render(request,'hospital/doctor_wait_for_approval.html')
    elif is_patient(request.user):
        accountapproval=models.Patient.objects.all().filter(user_id=request.user.id,status=True)
        if accountapproval:
            return redirect('patient-dashboard')
        else:
            return render(request,'hospital/patient_wait_for_approval.html')








#---------------------------------------------------------------------------------
#------------------------ ADMIN RELATED VIEWS START ------------------------------
#---------------------------------------------------------------------------------
@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_dashboard_view(request):
    #for both table in admin dashboard
    doctors=models.Doctor.objects.all().order_by('-id')
    patients=models.Patient.objects.all().order_by('-id')
    #for three cards
    doctorcount=models.Doctor.objects.all().filter(status=True).count()
    pendingdoctorcount=models.Doctor.objects.all().filter(status=False).count()

    patientcount=models.Patient.objects.all().filter(status=True).count()
    pendingpatientcount=models.Patient.objects.all().filter(status=False).count()

    appointmentcount=models.Appointment.objects.all().filter(status=True).count()
    pendingappointmentcount=models.Appointment.objects.all().filter(status=False).count()
    mydict={
    'doctors':doctors,
    'patients':patients,
    'doctorcount':doctorcount,
    'pendingdoctorcount':pendingdoctorcount,
    'patientcount':patientcount,
    'pendingpatientcount':pendingpatientcount,
    'appointmentcount':appointmentcount,
    'pendingappointmentcount':pendingappointmentcount,
    }
    return render(request,'hospital/admin_dashboard.html',context=mydict)


# this view for sidebar click on admin page
@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_doctor_view(request):
    return render(request,'hospital/admin_doctor.html')



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_view_doctor_view(request):
    doctors=models.Doctor.objects.all().filter(status=True)
    return render(request,'hospital/admin_view_doctor.html',{'doctors':doctors})



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def delete_doctor_from_hospital_view(request,pk):
    doctor=models.Doctor.objects.get(id=pk)
    user=models.User.objects.get(id=doctor.user_id)
    user.delete()
    doctor.delete()
    return redirect('admin-view-doctor')



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def update_doctor_view(request,pk):
    doctor=models.Doctor.objects.get(id=pk)
    user=models.User.objects.get(id=doctor.user_id)

    userForm=forms.DoctorUserForm(instance=user)
    doctorForm=forms.DoctorForm(request.FILES,instance=doctor)
    mydict={'userForm':userForm,'doctorForm':doctorForm}
    if request.method=='POST':
        userForm=forms.DoctorUserForm(request.POST,instance=user)
        doctorForm=forms.DoctorForm(request.POST,request.FILES,instance=doctor)
        if userForm.is_valid() and doctorForm.is_valid():
            user=userForm.save()
            user.set_password(user.password)
            user.save()
            doctor=doctorForm.save(commit=False)
            doctor.status=True
            doctor.save()
            return redirect('admin-view-doctor')
    return render(request,'hospital/admin_update_doctor.html',context=mydict)




@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_add_doctor_view(request):
    userForm=forms.DoctorUserForm()
    doctorForm=forms.DoctorForm()
    mydict={'userForm':userForm,'doctorForm':doctorForm}
    if request.method=='POST':
        userForm=forms.DoctorUserForm(request.POST)
        doctorForm=forms.DoctorForm(request.POST, request.FILES)
        if userForm.is_valid() and doctorForm.is_valid():
            user=userForm.save()
            user.set_password(user.password)
            user.save()

            doctor=doctorForm.save(commit=False)
            doctor.user=user
            doctor.status=True
            doctor.save()

            my_doctor_group = Group.objects.get_or_create(name='DOCTOR')
            my_doctor_group[0].user_set.add(user)

        return HttpResponseRedirect('admin-view-doctor')
    return render(request,'hospital/admin_add_doctor.html',context=mydict)




@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_approve_doctor_view(request):
    #those whose approval are needed
    doctors=models.Doctor.objects.all().filter(status=False)
    return render(request,'hospital/admin_approve_doctor.html',{'doctors':doctors})


@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def approve_doctor_view(request,pk):
    doctor=models.Doctor.objects.get(id=pk)
    doctor.status=True
    doctor.save()
    return redirect(reverse('admin-approve-doctor'))


@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def reject_doctor_view(request,pk):
    doctor=models.Doctor.objects.get(id=pk)
    user=models.User.objects.get(id=doctor.user_id)
    user.delete()
    doctor.delete()
    return redirect('admin-approve-doctor')



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_view_doctor_specialisation_view(request):
    doctors=models.Doctor.objects.all().filter(status=True)
    return render(request,'hospital/admin_view_doctor_specialisation.html',{'doctors':doctors})



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_patient_view(request):
    return render(request,'hospital/admin_patient.html')



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_view_patient_view(request):
    patients=models.Patient.objects.all().filter(status=True)
    return render(request,'hospital/admin_view_patient.html',{'patients':patients})



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def delete_patient_from_hospital_view(request,pk):
    patient=models.Patient.objects.get(id=pk)
    user=models.User.objects.get(id=patient.user_id)
    user.delete()
    patient.delete()
    return redirect('admin-view-patient')



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def update_patient_view(request,pk):
    patient=models.Patient.objects.get(id=pk)
    user=models.User.objects.get(id=patient.user_id)

    userForm=forms.PatientUserForm(instance=user)
    patientForm=forms.PatientForm(request.FILES,instance=patient)
    mydict={'userForm':userForm,'patientForm':patientForm}
    if request.method=='POST':
        userForm=forms.PatientUserForm(request.POST,instance=user)
        patientForm=forms.PatientForm(request.POST,request.FILES,instance=patient)
        if userForm.is_valid() and patientForm.is_valid():
            user=userForm.save()
            user.set_password(user.password)
            user.save()
            patient=patientForm.save(commit=False)
            patient.status=True
            patient.assignedDoctorId=request.POST.get('assignedDoctorId')
            patient.save()
            return redirect('admin-view-patient')
    return render(request,'hospital/admin_update_patient.html',context=mydict)





@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_add_patient_view(request):
    userForm=forms.PatientUserForm()
    patientForm=forms.PatientForm()
    mydict={'userForm':userForm,'patientForm':patientForm}
    if request.method=='POST':
        userForm=forms.PatientUserForm(request.POST)
        patientForm=forms.PatientForm(request.POST,request.FILES)
        if userForm.is_valid() and patientForm.is_valid():
            user=userForm.save()
            user.set_password(user.password)
            user.save()

            patient=patientForm.save(commit=False)
            patient.user=user
            patient.status=True
            patient.assignedDoctorId=request.POST.get('assignedDoctorId')
            patient.save()

            my_patient_group = Group.objects.get_or_create(name='PATIENT')
            my_patient_group[0].user_set.add(user)

        return HttpResponseRedirect('admin-view-patient')
    return render(request,'hospital/admin_add_patient.html',context=mydict)



#------------------FOR APPROVING PATIENT BY ADMIN----------------------
@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_approve_patient_view(request):
    #those whose approval are needed
    patients=models.Patient.objects.all().filter(status=False)
    return render(request,'hospital/admin_approve_patient.html',{'patients':patients})



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def approve_patient_view(request,pk):
    patient=models.Patient.objects.get(id=pk)
    patient.status=True
    patient.save()
    return redirect(reverse('admin-approve-patient'))



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def reject_patient_view(request,pk):
    patient=models.Patient.objects.get(id=pk)
    user=models.User.objects.get(id=patient.user_id)
    user.delete()
    patient.delete()
    return redirect('admin-approve-patient')



#--------------------- FOR DISCHARGING PATIENT BY ADMIN START-------------------------
@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_discharge_patient_view(request):
    patients=models.Patient.objects.all().filter(status=True)
    return render(request,'hospital/admin_discharge_patient.html',{'patients':patients})



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def discharge_patient_view(request,pk):
    patient=models.Patient.objects.get(id=pk)
    days=(date.today()-patient.admitDate) #2 days, 0:00:00
    assignedDoctor=models.User.objects.all().filter(id=patient.assignedDoctorId)
    d=days.days # only how many day that is 2
    patientDict={
        'patientId':pk,
        'name':patient.get_name,
        'mobile':patient.mobile,
        'address':patient.address,
        'symptoms':patient.symptoms,
        'admitDate':patient.admitDate,
        'todayDate':date.today(),
        'day':d,
        'assignedDoctorName':assignedDoctor[0].first_name,
    }
    if request.method == 'POST':
        feeDict ={
            'roomCharge':int(request.POST['roomCharge'])*int(d),
            'doctorFee':request.POST['doctorFee'],
            'medicineCost' : request.POST['medicineCost'],
            'OtherCharge' : request.POST['OtherCharge'],
            'total':(int(request.POST['roomCharge'])*int(d))+int(request.POST['doctorFee'])+int(request.POST['medicineCost'])+int(request.POST['OtherCharge'])
        }
        patientDict.update(feeDict)
        #for updating to database patientDischargeDetails (pDD)
        pDD=models.PatientDischargeDetails()
        pDD.patientId=pk
        pDD.patientName=patient.get_name
        pDD.assignedDoctorName=assignedDoctor[0].first_name
        pDD.address=patient.address
        pDD.mobile=patient.mobile
        pDD.symptoms=patient.symptoms
        pDD.admitDate=patient.admitDate
        pDD.releaseDate=date.today()
        pDD.daySpent=int(d)
        pDD.medicineCost=int(request.POST['medicineCost'])
        pDD.roomCharge=int(request.POST['roomCharge'])*int(d)
        pDD.doctorFee=int(request.POST['doctorFee'])
        pDD.OtherCharge=int(request.POST['OtherCharge'])
        pDD.total=(int(request.POST['roomCharge'])*int(d))+int(request.POST['doctorFee'])+int(request.POST['medicineCost'])+int(request.POST['OtherCharge'])
        pDD.save()
        return render(request,'hospital/patient_final_bill.html',context=patientDict)
    return render(request,'hospital/patient_generate_bill.html',context=patientDict)



#--------------for discharge patient bill (pdf) download and printing
import io
from xhtml2pdf import pisa
from django.template.loader import get_template
from django.template import Context
from django.http import HttpResponse


def render_to_pdf(template_src, context_dict):
    template = get_template(template_src)
    html  = template.render(context_dict)
    result = io.BytesIO()
    pdf = pisa.pisaDocument(io.BytesIO(html.encode("ISO-8859-1")), result)
    if not pdf.err:
        return HttpResponse(result.getvalue(), content_type='application/pdf')
    return



def download_pdf_view(request,pk):
    dischargeDetails=models.PatientDischargeDetails.objects.all().filter(patientId=pk).order_by('-id')[:1]
    dict={
        'patientName':dischargeDetails[0].patientName,
        'assignedDoctorName':dischargeDetails[0].assignedDoctorName,
        'address':dischargeDetails[0].address,
        'mobile':dischargeDetails[0].mobile,
        'symptoms':dischargeDetails[0].symptoms,
        'admitDate':dischargeDetails[0].admitDate,
        'releaseDate':dischargeDetails[0].releaseDate,
        'daySpent':dischargeDetails[0].daySpent,
        'medicineCost':dischargeDetails[0].medicineCost,
        'roomCharge':dischargeDetails[0].roomCharge,
        'doctorFee':dischargeDetails[0].doctorFee,
        'OtherCharge':dischargeDetails[0].OtherCharge,
        'total':dischargeDetails[0].total,
    }
    return render_to_pdf('hospital/download_bill.html',dict)



#-----------------APPOINTMENT START--------------------------------------------------------------------
@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_appointment_view(request):
    return render(request,'hospital/admin_appointment.html')



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_view_appointment_view(request):
    appointments=models.Appointment.objects.all().filter(status=True)
    return render(request,'hospital/admin_view_appointment.html',{'appointments':appointments})



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_add_appointment_view(request):
    appointmentForm=forms.AppointmentForm()
    mydict={'appointmentForm':appointmentForm,}
    if request.method=='POST':
        appointmentForm=forms.AppointmentForm(request.POST)
        if appointmentForm.is_valid():
            appointment=appointmentForm.save(commit=False)
            appointment.doctorId=request.POST.get('doctorId')
            appointment.patientId=request.POST.get('patientId')
            appointment.doctorName=models.User.objects.get(id=request.POST.get('doctorId')).first_name
            appointment.patientName=models.User.objects.get(id=request.POST.get('patientId')).first_name
            appointment.status=True
            appointment.save()
        return HttpResponseRedirect('admin-view-appointment')
    return render(request,'hospital/admin_add_appointment.html',context=mydict)



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def admin_approve_appointment_view(request):
    #those whose approval are needed
    appointments=models.Appointment.objects.all().filter(status=False)
    return render(request,'hospital/admin_approve_appointment.html',{'appointments':appointments})



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def approve_appointment_view(request,pk):
    appointment=models.Appointment.objects.get(id=pk)
    appointment.status=True
    appointment.save()
    return redirect(reverse('admin-approve-appointment'))



@login_required(login_url='adminlogin')
@user_passes_test(is_admin)
def reject_appointment_view(request,pk):
    appointment=models.Appointment.objects.get(id=pk)
    appointment.delete()
    return redirect('admin-approve-appointment')
#---------------------------------------------------------------------------------
#------------------------ ADMIN RELATED VIEWS END ------------------------------
#---------------------------------------------------------------------------------






#---------------------------------------------------------------------------------
#------------------------ DOCTOR RELATED VIEWS START ------------------------------
#---------------------------------------------------------------------------------
@login_required(login_url='doctorlogin')
@user_passes_test(is_doctor)
def doctor_dashboard_view(request):
    #for three cards
    patientcount=models.Patient.objects.all().filter(status=True,assignedDoctorId=request.user.id).count()
    appointmentcount=models.Appointment.objects.all().filter(status=True,doctorId=request.user.id).count()
    patientdischarged=models.PatientDischargeDetails.objects.all().distinct().filter(assignedDoctorName=request.user.first_name).count()

    #for  table in doctor dashboard
    appointments=models.Appointment.objects.all().filter(status=True,doctorId=request.user.id).order_by('-id')
    patientid=[]
    for a in appointments:
        patientid.append(a.patientId)
    patients=models.Patient.objects.all().filter(status=True,user_id__in=patientid).order_by('-id')
    appointments=zip(appointments,patients)
    mydict={
    'patientcount':patientcount,
    'appointmentcount':appointmentcount,
    'patientdischarged':patientdischarged,
    'appointments':appointments,
    'doctor':models.Doctor.objects.get(user_id=request.user.id), #for profile picture of doctor in sidebar
    }
    return render(request,'hospital/doctor_dashboard.html',context=mydict)



@login_required(login_url='doctorlogin')
@user_passes_test(is_doctor)
def doctor_patient_view(request):
    mydict={
    'doctor':models.Doctor.objects.get(user_id=request.user.id), #for profile picture of doctor in sidebar
    }
    return render(request,'hospital/doctor_patient.html',context=mydict)



@login_required(login_url='doctorlogin')
@user_passes_test(is_doctor)
def doctor_view_patient_view(request):
    patients=models.Patient.objects.all().filter(status=True,assignedDoctorId=request.user.id)
    doctor=models.Doctor.objects.get(user_id=request.user.id) #for profile picture of doctor in sidebar
    return render(request,'hospital/doctor_view_patient.html',{'patients':patients,'doctor':doctor})



@login_required(login_url='doctorlogin')
@user_passes_test(is_doctor)
def doctor_view_discharge_patient_view(request):
    dischargedpatients=models.PatientDischargeDetails.objects.all().distinct().filter(assignedDoctorName=request.user.first_name)
    doctor=models.Doctor.objects.get(user_id=request.user.id) #for profile picture of doctor in sidebar
    return render(request,'hospital/doctor_view_discharge_patient.html',{'dischargedpatients':dischargedpatients,'doctor':doctor})



@login_required(login_url='doctorlogin')
@user_passes_test(is_doctor)
def doctor_appointment_view(request):
    doctor=models.Doctor.objects.get(user_id=request.user.id) #for profile picture of doctor in sidebar
    return render(request,'hospital/doctor_appointment.html',{'doctor':doctor})



@login_required(login_url='doctorlogin')
@user_passes_test(is_doctor)
def doctor_view_appointment_view(request):
    doctor=models.Doctor.objects.get(user_id=request.user.id) #for profile picture of doctor in sidebar
    appointments=models.Appointment.objects.all().filter(status=True,doctorId=request.user.id)
    patientid=[]
    for a in appointments:
        patientid.append(a.patientId)
    patients=models.Patient.objects.all().filter(status=True,user_id__in=patientid)
    appointments=zip(appointments,patients)
    return render(request,'hospital/doctor_view_appointment.html',{'appointments':appointments,'doctor':doctor})



@login_required(login_url='doctorlogin')
@user_passes_test(is_doctor)
def doctor_delete_appointment_view(request):
    doctor=models.Doctor.objects.get(user_id=request.user.id) #for profile picture of doctor in sidebar
    appointments=models.Appointment.objects.all().filter(status=True,doctorId=request.user.id)
    patientid=[]
    for a in appointments:
        patientid.append(a.patientId)
    patients=models.Patient.objects.all().filter(status=True,user_id__in=patientid)
    appointments=zip(appointments,patients)
    return render(request,'hospital/doctor_delete_appointment.html',{'appointments':appointments,'doctor':doctor})



@login_required(login_url='doctorlogin')
@user_passes_test(is_doctor)
def delete_appointment_view(request,pk):
    appointment=models.Appointment.objects.get(id=pk)
    appointment.delete()
    doctor=models.Doctor.objects.get(user_id=request.user.id) #for profile picture of doctor in sidebar
    appointments=models.Appointment.objects.all().filter(status=True,doctorId=request.user.id)
    patientid=[]
    for a in appointments:
        patientid.append(a.patientId)
    patients=models.Patient.objects.all().filter(status=True,user_id__in=patientid)
    appointments=zip(appointments,patients)
    return render(request,'hospital/doctor_delete_appointment.html',{'appointments':appointments,'doctor':doctor})



#---------------------------------------------------------------------------------
#------------------------ DOCTOR RELATED VIEWS END ------------------------------
#---------------------------------------------------------------------------------






#---------------------------------------------------------------------------------
#------------------------ PATIENT RELATED VIEWS START ------------------------------
#---------------------------------------------------------------------------------
@login_required(login_url='patientlogin')
@user_passes_test(is_patient)
def patient_dashboard_view(request):
    patient=models.Patient.objects.get(user_id=request.user.id)
    doctor=models.Doctor.objects.get(user_id=patient.assignedDoctorId)
    mydict={
    'patient':patient,
    'doctorName':doctor.get_name,
    'doctorMobile':doctor.mobile,
    'doctorAddress':doctor.address,
    'symptoms':patient.symptoms,
    'doctorDepartment':doctor.department,
    'admitDate':patient.admitDate,
    }
    return render(request,'hospital/patient_dashboard.html',context=mydict)



@login_required(login_url='patientlogin')
@user_passes_test(is_patient)
def patient_appointment_view(request):
    patient=models.Patient.objects.get(user_id=request.user.id) #for profile picture of patient in sidebar
    return render(request,'hospital/patient_appointment.html',{'patient':patient})



@login_required(login_url='patientlogin')
@user_passes_test(is_patient)
def patient_book_appointment_view(request):
    appointmentForm=forms.PatientAppointmentForm()
    patient=models.Patient.objects.get(user_id=request.user.id) #for profile picture of patient in sidebar
    mydict={'appointmentForm':appointmentForm,'patient':patient}
    if request.method=='POST':
        appointmentForm=forms.PatientAppointmentForm(request.POST)
        if appointmentForm.is_valid():
            appointment=appointmentForm.save(commit=False)
            appointment.doctorId=request.POST.get('doctorId')
            appointment.patientId=request.user.id #----user can choose any patient but only their info will be stored
            appointment.doctorName=models.User.objects.get(id=request.POST.get('doctorId')).first_name
            appointment.patientName=request.user.first_name #----user can choose any patient but only their info will be stored
            appointment.status=False
            appointment.save()
        return HttpResponseRedirect('patient-view-appointment')
    return render(request,'hospital/patient_book_appointment.html',context=mydict)





@login_required(login_url='patientlogin')
@user_passes_test(is_patient)
def patient_view_appointment_view(request):
    patient=models.Patient.objects.get(user_id=request.user.id) #for profile picture of patient in sidebar
    appointments=models.Appointment.objects.all().filter(patientId=request.user.id)
    return render(request,'hospital/patient_view_appointment.html',{'appointments':appointments,'patient':patient})



@login_required(login_url='patientlogin')
@user_passes_test(is_patient)
def patient_discharge_view(request):
    patient=models.Patient.objects.get(user_id=request.user.id) #for profile picture of patient in sidebar
    dischargeDetails=models.PatientDischargeDetails.objects.all().filter(patientId=patient.id).order_by('-id')[:1]
    patientDict=None
    if dischargeDetails:
        patientDict ={
        'is_discharged':True,
        'patient':patient,
        'patientId':patient.id,
        'patientName':patient.get_name,
        'assignedDoctorName':dischargeDetails[0].assignedDoctorName,
        'address':patient.address,
        'mobile':patient.mobile,
        'symptoms':patient.symptoms,
        'admitDate':patient.admitDate,
        'releaseDate':dischargeDetails[0].releaseDate,
        'daySpent':dischargeDetails[0].daySpent,
        'medicineCost':dischargeDetails[0].medicineCost,
        'roomCharge':dischargeDetails[0].roomCharge,
        'doctorFee':dischargeDetails[0].doctorFee,
        'OtherCharge':dischargeDetails[0].OtherCharge,
        'total':dischargeDetails[0].total,
        }
        print(patientDict)
    else:
        patientDict={
            'is_discharged':False,
            'patient':patient,
            'patientId':request.user.id,
        }
    return render(request,'hospital/patient_discharge.html',context=patientDict)


#------------------------ PATIENT RELATED VIEWS END ------------------------------
#---------------------------------------------------------------------------------








#---------------------------------------------------------------------------------
#------------------------ ABOUT US AND CONTACT US VIEWS START ------------------------------
#---------------------------------------------------------------------------------
def aboutus_view(request):
    return render(request,'hospital/aboutus.html')

def contactus_view(request):
    sub = forms.ContactusForm()
    if request.method == 'POST':
        sub = forms.ContactusForm(request.POST)
        if sub.is_valid():
            email = sub.cleaned_data['Email']
            name=sub.cleaned_data['Name']
            message = sub.cleaned_data['Message']
            send_mail(str(name)+' || '+str(email),message, EMAIL_HOST_USER, EMAIL_RECEIVING_USER, fail_silently = False)
            return render(request, 'hospital/contactussuccess.html')
    return render(request, 'hospital/contactus.html', {'form':sub})


#---------------------------------------------------------------------------------
#------------------------ ADMIN RELATED VIEWS END ------------------------------
#---------------------------------------------------------------------------------



#Developed By : alamin
#facebook : fb.com/msacom.tk
#Youtube :youtube.com/lazycoders



"""
WSGI config for hospitalmanagement project.

It exposes the WSGI callable as a module-level variable named ``application``.

For more information on this file, see
https://docs.djangoproject.com/en/3.0/howto/deployment/wsgi/
"""

import os

from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'hospitalmanagement.settings')

application = get_wsgi_application()



body {

  padding-left: 240px;
}
main {
  position: relative;
  height: 100vh;
}

.menu {
  background: #5bc995;
  height: 100vh;
  width: 240px;
  position: fixed;
  top: 0px;
  left: 0;
  z-index: 5;
  outline: none;
}
.menu .avatar {
  background: rgba(0, 0, 0, 0.1);
  padding: 2em 0.5em;
  text-align: center;
}
.menu .avatar img {
  width: 100px;
  border-radius: 50%;
  overflow: hidden;
  border: 4px solid #ffea92;
  box-shadow: 0 0 0 4px rgba(255, 255, 255, 0.2);
}
.menu .avatar h2 {
  font-weight: normal;
  margin-bottom: 0;
}
.menu ul {
  list-style: none;
  padding: 0.5em 0;
  margin: 0;
}
.menu ul li {
  padding: 0.5em 1em 0.5em 3em;
  font-size: 0.95em;
  font-weight: regular;
  background-repeat: no-repeat;
  background-position: left 15px center;
  background-size: auto 20px;
  transition: all 0.15s linear;
  cursor: pointer;
}
.menu ul li.icon-dashboard {
  background-image: url("http://www.entypo.com/images//gauge.svg");
}
.menu ul li.icon-customers {
  background-image: url("http://www.entypo.com/images//briefcase.svg");
}
.menu ul li.icon-users {
  background-image: url("http://www.entypo.com/images//users.svg");
}
.menu ul li.icon-calendar {
  background-image: url("http://www.entypo.com/images//calendar.svg");
}

.menu ul li:hover {
  background-color: rgba(0, 0, 0, 0.1);
}
.menu ul li:focus {
  outline: none;
}
@media screen and (max-width: 900px) and (min-width: 400px) {
  body {
    padding-left: 90px;
  }
  .menu {
    width: 90px;
  }
  .menu .avatar {
    padding: 0.5em;
    position: relative;
  }
  .menu .avatar img {
    width: 60px;
  }
  .menu .avatar h2 {
    opacity: 0;
    position: absolute;
    top: 50%;
    left: 100px;
    margin: 0;
    min-width: 200px;
    border-radius: 4px;
    background: rgba(0, 0, 0, 0.4);
    transform: translate3d(-20px, -50%, 0);
    transition: all 0.15s ease-in-out;
  }
  .menu .avatar:hover h2 {
    opacity: 1;
    transform: translate3d(0px, -50%, 0);
  }
  .menu ul li {
    height: 60px;
    background-position: center center;
    background-size: 30px auto;
    position: relative;
  }
  .menu ul li span {
    opacity: 0;
    position: absolute;
    background: rgba(0, 0, 0, 0.5);
    padding: 0.2em 0.5em;
    border-radius: 4px;
    top: 50%;
    left: 80px;
    transform: translate3d(-15px, -50%, 0);
    transition: all 0.15s ease-in-out;
  }
  .menu ul li span:before {
    content: '';
    width: 0;
    height: 0;
    position: absolute;
    top: 50%;
    left: -5px;
    border-top: 5px solid transparent;
    border-bottom: 5px solid transparent;
    border-right: 5px solid rgba(0, 0, 0, 0.5);
    transform: translateY(-50%);
  }
  .menu ul li:hover span {
    opacity: 1;
    transform: translate3d(0px, -50%, 0);
  }
}
@media screen and (max-width: 400px) {
  body {
    padding-left: 0;
  }
  .menu {
    width: 230px;
    box-shadow: 0 0 0 100em rgba(0, 0, 0, 0);
    transform: translate3d(-230px, 0, 0);
    transition: all 0.3s ease-in-out;
  }
  .menu .smartphone-menu-trigger {
    width: 40px;
    height: 40px;
    position: absolute;
    left: 100%;
    background: #5bc995;
  }
  .menu .smartphone-menu-trigger:before,
  .menu .smartphone-menu-trigger:after {
    content: '';
    width: 50%;
    height: 2px;
    background: #fff;
    border-radius: 10px;
    position: absolute;
    top: 45%;
    left: 50%;
    transform: translate3d(-50%, -50%, 0);
  }
  .menu .smartphone-menu-trigger:after {
    top: 55%;
    transform: translate3d(-50%, -50%, 0);
  }
  .menu ul li {
    padding: 1em 1em 1em 3em;
    font-size: 1.2em;
  }
  .menu:focus {
    transform: translate3d(0, 0, 0);
    box-shadow: 0 0 0 100em rgba(0, 0, 0, 0.6);
  }
  .menu:focus .smartphone-menu-trigger {
    pointer-events: none;
  }
}




<!DOCTYPE html>
{% load static %}
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  <title>LazyCoder || sumit</title>
  <style media="screen">
    .jumbotron {
      margin-top: 0px;
      margin-bottom: 0px;
    }

    .jumbotron h1 {
      text-align: center;
    }

    .alert {
      margin: 0px;
    }
  </style>
  <title>sumit</title>
</head>

<body>
  {% include "hospital/navbar.html" %}
  <br><br>
  <center>
    <h3 class='alert alert-success' style="margin-bottom:0px;">About Us !</h3>
  </center>
  <div class="jumbotron" style="margin-bottom: 0px;margin-top: 0px;">
    <h1 class="display-4">Hello</h1>
    <p class="lead">A service dedicated to Hospital Admin, Doctor and Patient.</p>
    <hr class="my-4">
    <p>Explore our Website.</p>
    <p class="lead">
      <a class="btn btn-primary btn-lg" href="/" role="button">HOME</a>
    </p>
  </div>
  {% include "hospital/footer.html" %}
</body>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>



{% extends 'hospital/admin_base.html' %}
{% load widget_tweaks %}
{% block content %}

<head>
  <style media="screen">
    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }

    .menu {
      top: 50px;
    }
  </style>

  <link href="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/js/bootstrap.min.js"></script>
  <script src="//cdnjs.cloudflare.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>
</head>
<br><br>
<!------ add appointment page by admin(sumit)  ---------->
<form method="post">
  {% csrf_token %}
  <div class="container register-form">
    <div class="form">
      <div class="note">
        <p>Book Appointment Details</p>
      </div>
      <div class="form-content">
        <div class="row">
          <div class="col-md-12">
            <div class="form-group">
              {% render_field appointmentForm.description class="form-control" placeholder="Description" %}
            </div>
            <div class="form-group">
              {% render_field appointmentForm.doctorId class="form-control" placeholder="doctor" %}
            </div>
            <div class="form-group">
              {% render_field appointmentForm.patientId class="form-control" placeholder="patient" %}
            </div>

          </div>

        </div>
        <button type="submit" class="btnSubmit">Book</button>
      </div>
    </div>
  </div>
</form>

<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}


{% extends 'hospital/admin_base.html' %}
{% load widget_tweaks %}
{% block content %}

<head>
  <style media="screen">
    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }

    .menu {
      top: 50px;
    }
  </style>

  <link href="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/js/bootstrap.min.js"></script>
  <script src="//cdnjs.cloudflare.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>

</head>
<br><br><br>
<!------ signup page for doctor by admin(sumit)  ---------->
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  <div class="container register-form">
    <div class="form">
      <div class="note">
        <p>Add New Doctor To Hospital</p>
      </div>
      <div class="form-content">
        <div class="row">
          <div class="col-md-6">
            <div class="form-group">
              {% render_field userForm.first_name class="form-control" placeholder="First Name" %}
            </div>
            <div class="form-group">
              {% render_field userForm.username class="form-control" placeholder="Username" %}
            </div>
            <div class="form-group">
              {% render_field doctorForm.department class="form-control" placeholder="Department" %}
            </div>
            <div class="form-group">
              {% render_field doctorForm.address class="form-control" placeholder="Address" %}
            </div>

          </div>
          <div class="col-md-6">
            <div class="form-group">
              {% render_field userForm.last_name class="form-control" placeholder="Last Name" %}
            </div>
            <div class="form-group">
              {% render_field userForm.password class="form-control" placeholder="Password" %}
            </div>
            <div class="form-group">
              {% render_field doctorForm.mobile class="form-control"  placeholder="Mobile" %}
            </div>
            <div class="form-group">
              {% render_field doctorForm.profile_pic required="required" class="form-control" placeholder="Profile Picture" %}
            </div>
          </div>
        </div>
        <button type="submit" class="btnSubmit">Register</button>
      </div>
    </div>
  </div>

</form>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->
{% endblock content %}



{% extends 'hospital/admin_base.html' %}
{% load widget_tweaks %}
{% block content %}

<head>
  <style media="screen">
    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }

    .menu {
      top: 50px;
    }
  </style>

  <link href="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/js/bootstrap.min.js"></script>
  <script src="//cdnjs.cloudflare.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>

</head>
<br><br><br>
<!------ signup page for doctor by admin(sumit)  ---------->
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  <div class="container register-form">
    <div class="form">
      <div class="note">
        <p>Admit Patient To Hospital</p>
      </div>
      <div class="form-content">
        <div class="row">
          <div class="col-md-6">
            <div class="form-group">
              {% render_field userForm.first_name class="form-control" placeholder="First Name" %}
            </div>
            <div class="form-group">
              {% render_field userForm.username class="form-control" placeholder="Username" %}
            </div>
            <div class="form-group">
              {% render_field patientForm.address class="form-control" placeholder="Address" %}
            </div>
            <div class="form-group">
              {% render_field patientForm.symptoms class="form-control" placeholder="Symptoms" %}
            </div>
            <div class="form-group">
              {% render_field patientForm.profile_pic required="required" class="form-control" placeholder="Profile Picture" %}
            </div>
          </div>
          <div class="col-md-6">
            <div class="form-group">
              {% render_field userForm.last_name class="form-control" placeholder="Last Name" %}
            </div>
            <div class="form-group">
              {% render_field userForm.password class="form-control" placeholder="Password" %}
            </div>
            <div class="form-group">
              {% render_field patientForm.mobile  class="form-control" placeholder="Mobile" %}
            </div>
            <div class="form-group">
              {% render_field patientForm.assignedDoctorId class="form-control" placeholder="Doctor" %}
            </div>
          </div>
        </div>
        <button type="submit" class="btnSubmit">Admit</button>
      </div>
    </div>
  </div>

</form>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->
{% endblock content %}



{% extends 'hospital/admin_base.html' %}
{% load static %}
{% block content %}
<br><br>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
  <link href="http://netdna.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.12.1/css/all.min.css">
  <style type="text/css">
    a:link {
      text-decoration: none;
    }

    .menu {
      top: 50px;
    }

    h6 {
      color: white;
    }

    .order-card {
      color: #fff;
    }

    .bg-c-blue {
      background: linear-gradient(45deg, #4099ff, #73b4ff);
    }

    .bg-c-green {
      background: linear-gradient(45deg, #2ed8b6, #59e0c5);
    }

    .bg-c-yellow {
      background: linear-gradient(45deg, #FFB64D, #ffcb80);
    }

    .bg-c-pink {
      background: linear-gradient(45deg, #FF5370, #ff869a);
    }


    .card {
      border-radius: 5px;
      -webkit-box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      border: none;
      margin-bottom: 30px;
      -webkit-transition: all 0.3s ease-in-out;
      transition: all 0.3s ease-in-out;
    }

    .card .card-block {
      padding: 25px;
    }

    .order-card i {
      font-size: 26px;
    }

    .f-left {
      float: left;
    }

    .f-right {
      float: right;
    }
  </style>
</head>

<link href="https://maxcdn.bootstrapcdn.com/font-awesome/4.3.0/css/font-awesome.min.css" rel="stylesheet">
<div class="container">
  <div class="row">
    <div class="col-md-4 col-xl-4">
      <div class="card bg-c-blue order-card">
        <div class="card-block">
          <a href="/admin-view-appointment">
            <h6 class="m-b-20">View Appointment</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-calendar f-left"></i></h2>

        </div>
      </div>
    </div>

    <div class="col-md-4 col-xl-4">
      <div class="card bg-c-green order-card">
        <div class="card-block">
          <a href="/admin-add-appointment">
            <h6 class="m-b-20">Book Appointment</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-book-medical f-left"></i></h2>
        </div>
      </div>
    </div>

    <div class="col-md-4 col-xl-4">
      <div class="card bg-c-yellow order-card">
        <div class="card-block">
          <a href="/admin-approve-appointment">
            <h6 class="m-b-20">Approve Appointment</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-check-circle f-left"></i></h2>
        </div>
      </div>
    </div>


  </div>
</div>
<!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->
{% endblock content %}


{% extends 'hospital/admin_base.html' %}
{% block content %}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>
  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Appointment Approvals Required</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Doctor Name</th>
          <th>Patient Name</th>
          <th>Description</th>
          <th>Date</th>
          <th>Approve</th>
          <th>Reject</th>
        </tr>
      </thead>
      {% for a in appointments %}
      <tr>
        <td> {{a.doctorName}}</td>
        <td>{{a.patientName}}</td>
        <td>{{a.description}}</td>
        <td>{{a.appointmentDate}}</td>
        <td><a class="btn btn-primary btn-xs" href="{% url 'approve-appointment' a.id  %}"><span class="glyphicon glyphicon-ok"></span></a></td>
        <td><a class="btn btn-danger btn-xs" href="{% url 'reject-appointment' a.id  %}"><span class="glyphicon glyphicon-trash"></span></a></td>
      </tr>
      {% endfor %}
    </table>
  </div>


</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}


{% extends 'hospital/admin_base.html' %}
{% block content %}
{%load static%}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>
  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Doctors &nbsp Applied For Registration</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Name</th>
          <th>Profile Picture</th>
          <th>Mobile</th>
          <th>Address</th>
          <th>Department</th>
          <th>Approve</th>
          <th>Reject</th>
        </tr>
      </thead>
      {% for d in doctors %}
      <tr>
        <td> {{d.get_name}}</td>
        <td> <img src="{% static d.profile_pic.url %}" alt="Profile Pic" height="40px" width="40px" /></td>
        <td>{{d.mobile}}</td>
        <td>{{d.address}}</td>
        <td>{{d.department}}</td>
        <td><a class="btn btn-primary btn-xs" href="{% url 'approve-doctor' d.id  %}"><span class="glyphicon glyphicon-ok"></span></a></td>
        <td><a class="btn btn-danger btn-xs" href="{% url 'reject-doctor' d.id  %}"><span class="glyphicon glyphicon-trash"></span></a></td>
      </tr>
      {% endfor %}
    </table>
  </div>


</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}


{% extends 'hospital/admin_base.html' %}
{% block content %}
{%load static%}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>
  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Patient Wants To Admit</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Name</th>
          <th>Profile Picture</th>
          <th>Symptoms</th>
          <th>Mobile</th>
          <th>Address</th>
          <th>Approve</th>
          <th>Reject</th>
        </tr>
      </thead>
      {% for p in patients %}
      <tr>
        <td> {{p.get_name}}</td>
        <td> <img src="{% static p.profile_pic.url %}" alt="Profile Pic" height="40px" width="40px" /></td>
        <td>{{p.symptoms}}</td>
        <td>{{p.mobile}}</td>
        <td>{{p.address}}</td>
        <td><a class="btn btn-primary btn-xs" href="{% url 'approve-patient' p.id  %}"><span class="glyphicon glyphicon-ok"></span></a></td>
        <td><a class="btn btn-danger btn-xs" href="{% url 'reject-patient' p.id  %}"><span class="glyphicon glyphicon-trash"></span></a></td>
      </tr>
      {% endfor %}
    </table>
  </div>


</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}


<!DOCTYPE html>
{% load static %}
<html lang="en">

<head>
  <meta charset="UTF-8">
  <title>LazyCoder || sumit</title>

  <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css">
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/font-awesome/4.7.0/css/font-awesome.min.css">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js"></script>
  <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js"></script>

  <link rel="stylesheet" href="{% static '/style.css' %}">


  <style media="screen">
    a:link {
      text-decoration: none;
    }

    a {
      color: white;
    }

    a:hover {
      color: white;
    }




    /*---------------------------------------
       Social section
    -----------------------------------------*/
    footer {
      padding: 0px 0px 0px 0px;
      background-color: black;
      margin: 0px;
    }


    #ftr {

      padding: 20px;
    }

    .fa {

      font-size: 23px;
      width: 60px;
      text-align: center;
      text-decoration: none;
      margin: 5px 2px;
      border-radius: 50%;
    }

    .fa:hover {
      opacity: 0.5;
      text-decoration: none;
    }

    .fa-facebook {
      background: #3B5998;
      color: white;
      margin-top: 30px;
    }

    .fa-whatsapp {
      background: #25d366;
      color: white;
    }

    .fa-twitter {
      background: #55ACEE;
      color: white;
    }

    .fa-instagram {
      background: #125688;
      color: white;
    }

    p {
      text-align: center;

    }
  </style>

</head>

<body>

  <!-- partial:index.partial.html -->
  <nav class="menu" tabindex="0">
    <div class="smartphone-menu-trigger"></div>
    <header class="avatar">
      <img src="{% static "images/adminpropic.png" %}" />
      <br><br>
      <h6>Admin</h6>
      <h2>{{request.user.first_name}}</h2>
    </header>
    <ul>
      <li tabindex="0" class="icon-dashboard"> <a style="color:white; text-decoration:none;" href="/admin-dashboard"><span>Dashboard</span></a> </li>
      <li tabindex="0" class="icon-customers"> <a style="color:white; text-decoration:none;" href="/admin-doctor"><span>Doctor</span></a></li>
      <li tabindex="0" class="icon-users"> <a style="color:white; text-decoration:none;" href="/admin-patient"><span>Patient</span></a></li>
      <li tabindex="0" class="icon-calendar"> <a style="color:white; text-decoration:none;" href="/admin-appointment"><span>Appointment</span></a></li>
    </ul>
  </nav>

  <main>


    <!-- nav start -->
    <div class="bs-example">
      <nav class="navbar navbar-expand-md  navbar-dark fixed-top" style="background:#337AB7;">
        <a href="/admin-dashboard" class="navbar-brand">HOSPITAL MANAGEMENT</a>
        <button type="button" class="navbar-toggler" data-toggle="collapse" data-target="#navbarCollapse">
          <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse justify-content-between" id="navbarCollapse">
          <div class="navbar-nav" style=" margin-left: 90%;">

            <a href="/logout" class="nav-item nav-link">Logout</a>
          </div>
        </div>
      </nav>
    </div>
    <!-- nav end -->



    <br><br>

    <!-- content start-->
    {% block content %}

    {% endblock content %}
    <!-- content end-->

    <br><br><br><br>
    <footer>
      <p>
        <a id="ftr" href="https://facebook.com/sumit.luv/" class="fa fa-facebook"></a>
        <a id="ftr" href="https://api.whatsapp.com/send?phone=919572181024&text=Hello%20Sumit.%0d%0aHow%20are%20you%20%3f%0d%0aI%20came%20from%20your%20website.&source=&data=#" class="fa fa-whatsapp"></a>
        <a id="ftr" href="https://instagram.com/sumit.luv" class="fa fa-instagram"></a>
        <a id="ftr" href="https://twitter.com/sumitkumar1503" class="fa fa-twitter"></a>
      </p>

      <br>
      <div class="container">
        <div class="row">
          <div class="col-md-12 col-sm-12">
            <div style="color:#ffffff;" class="wow fadeInUp footer-copyright">
              <p>Made in India <br>
                Copyright &copy; 2020 LazyCoder </p>
            </div>
          </div>
        </div>
      </div>
    </footer>

  </main>
  <!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

</body>

</html>



{% extends 'hospital/admin_base.html' %}
{% load static %}
{% block content %}
{%include 'hospital/admin_dashboard_cards.html'%}
<br><br><br><br>
<div class="container">
  <div class="row">
    <div class="panel panel-primary col-md-5" style="margin-left:2%;">
      <div class="panel-heading" style="text-align:center;">
        <h6 class="panel-title">Recent Doctors</h6>
      </div>
      <table class="table table-hover" id="dev-table">
        <thead>
          <tr>
            <th>Name</th>
            <th>Department</th>
            <th>Mobile</th>
            <th>Status</th>

          </tr>
        </thead>
        {% for d in doctors %}
        <tr>
          <td> {{d.get_name}}</td>
          <td>{{d.department}}</td>
          <td>{{d.mobile}}</td>
          {%if d.status%}
          <td> <span class="label label-primary">Permanent</span></td>
          {% else %}
          <td> <span class="label label-success">On Hold</span></td>
          {% endif %}

        </tr>
        {% endfor %}
      </table>
    </div>

    <div class="panel panel-primary col-md-5" style="margin-left:5%;">
      <div class="panel-heading" style="text-align:center;">
        <h6 class="panel-title">Recent Patient</h6>
      </div>
      <table class="table table-hover" id="dev-table">
        <thead>
          <tr>
            <th>Name</th>
            <th>Symptoms</th>
            <th>Mobile</th>
            <th>Address</th>
            <th>Status</th>

          </tr>
        </thead>
        {% for p in patients %}
        <tr>
          <td> {{p.get_name}}</td>
          <td>{{p.symptoms}}</td>
          <td>{{p.mobile}}</td>
          <td>{{p.address}}</td>
          {%if p.status%}
          <td> <span class="label label-primary">Admitted</span></td>
          {% else %}
          <td> <span class="label label-success">On Hold</span></td>
          {% endif %}

        </tr>
        {% endfor %}
      </table>
    </div>
  </div>
</div>

<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}



<!DOCTYPE html>
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <title></title>

  <style media="screen">
    .market-update-block {
      padding: 2em 2em;
      background: #999;
    }

    .market-update-block h3 {
      color: #fff;
      font-size: 2.5em;
      font-family: 'Carrois Gothic', sans-serif;
    }

    .market-update-block h4 {
      font-size: 1.2em;
      color: #fff;
      margin: 0.3em 0em;
      font-family: 'Carrois Gothic', sans-serif;
    }

    .market-update-block p {
      color: #fff;
      font-size: 0.8em;
      line-height: 1.8em;
    }

    .market-update-block.clr-block-1 {
      background: #ff0000;
      margin-right: 0.8em;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-2 {
      background: #FC8213;
      margin-right: 0.8em;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-3 {
      background: #1355f9;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-1:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-2:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-3:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-right i.fa.fa-user-o {
      font-size: 3em;
      color: #68AE00;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-right i.fa.fa-user-md {
      font-size: 3em;
      color: #FC8213;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-right i.fa.fa-calendar {
      font-size: 3em;
      color: #337AB7;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-left {
      padding: 0px;
    }
  </style>
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
  <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/js/bootstrap.min.js"></script>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">
</head>

<body>
  <div class="market-updates">
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-1">
        <div class="col-md-8 market-update-left">
          <h3>{{doctorcount}}</h3>
          <h4>Total Doctor</h4>
          <p>Approval Required : {{pendingdoctorcount}}</p>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-user-md"></i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-2">
        <div class="col-md-8 market-update-left">
          <h3>{{patientcount}}</h3>
          <h4>Total Patient</h4>
          <p>Wants to Admit : {{pendingpatientcount}}</p>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-user-o"></i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-3">
        <div class="col-md-8 market-update-left">
          <h3>{{appointmentcount}}</h3>
          <h4>Total Appointment</h4>
          <p>Approve Appointments :{{pendingappointmentcount}} </p>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-calendar"> </i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="clearfix"> </div>
  </div>
</body>
<!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

</html>


{% extends 'hospital/admin_base.html' %}
{% block content %}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>
  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Discharge Patient</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Name</th>
          <th>Symptoms</th>
          <th>Mobile</th>
          <th>Discharge</th>
        </tr>
      </thead>
      {% for p in patients %}
      <tr>
        <td> {{p.get_name}}</td>
        <td>{{p.symptoms}}</td>
        <td>{{p.mobile}}</td>
        <td><a class="btn btn-primary btn-xs" href="{% url 'discharge-patient' p.id  %}"><span class="glyphicon glyphicon-log-out"></span></a></td>
      </tr>
      {% endfor %}
    </table>
  </div>


</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}



{% extends 'hospital/admin_base.html' %}
{% load static %}
{% block content %}
<br><br>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
  <link href="http://netdna.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet">

  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.12.1/css/all.min.css">

  <style type="text/css">
    a:link {
      text-decoration: none;
    }

    .menu {
      top: 50px;
    }

    h6 {
      color: white;
    }

    .order-card {
      color: #fff;
    }

    .bg-c-blue {
      background: linear-gradient(45deg, #4099ff, #73b4ff);
    }

    .bg-c-green {
      background: linear-gradient(45deg, #2ed8b6, #59e0c5);
    }

    .bg-c-yellow {
      background: linear-gradient(45deg, #FFB64D, #ffcb80);
    }

    .bg-c-pink {
      background: linear-gradient(45deg, #FF5370, #ff869a);
    }


    .card {
      border-radius: 5px;
      -webkit-box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      border: none;
      margin-bottom: 30px;
      -webkit-transition: all 0.3s ease-in-out;
      transition: all 0.3s ease-in-out;
    }

    .card .card-block {
      padding: 25px;
    }

    .order-card i {
      font-size: 26px;
    }

    .f-left {
      float: left;
    }

    .f-right {
      float: right;
    }
  </style>
</head>

<link href="https://maxcdn.bootstrapcdn.com/font-awesome/4.3.0/css/font-awesome.min.css" rel="stylesheet">
<div class="container">
  <div class="row">
    <div class="col-md-4 col-xl-3">
      <div class="card bg-c-blue order-card">
        <div class="card-block">
          <a href="/admin-view-doctor">
            <h6 class="m-b-20">Doctor Record</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-user-nurse f-left"></i></h2>

        </div>
      </div>
    </div>

    <div class="col-md-4 col-xl-3">
      <div class="card bg-c-green order-card">
        <div class="card-block">
          <a href="/admin-add-doctor">
            <h6 class="m-b-20">Register Doctor</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-user-plus f-left"></i></h2>
        </div>
      </div>
    </div>

    <div class="col-md-4 col-xl-3">
      <div class="card bg-c-yellow order-card">
        <div class="card-block">
          <a href="/admin-approve-doctor">
            <h6 class="m-b-20">Approve Doctor</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-check-circle f-left"></i></h2>
        </div>
      </div>
    </div>

    <div class="col-md-4 col-xl-3">
      <div class="card bg-c-pink order-card">
        <div class="card-block">
          <a href="/admin-view-doctor-specialisation">
            <h6 class="m-b-20">Doctor Specialisation</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-user-md f-left"></i></h2>
        </div>
      </div>
    </div>
  </div>
</div>
<!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

{% endblock content %}


<!DOCTYPE html>
{% load static %}
<html>

<head>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">
  <style>
    .card {
      box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2);
      max-width: 300px;
      margin: auto;
      text-align: center;
      font-family: arial;
    }

    .title {
      color: grey;
      font-size: 18px;
    }

    button {
      border: none;
      outline: 0;
      display: inline-block;
      padding: 8px;
      color: white;
      background-color: #000;
      text-align: center;
      cursor: pointer;
      width: 100%;
      font-size: 18px;
    }

    button:hover,
    a:hover {
      opacity: 0.7;
    }

    .grid-container {
      display: grid;
      grid-template-columns: auto auto auto;
      padding: 10px;
    }

    a:link {
      text-decoration: none;
    }

    a {
      color: white;
    }
  </style>
</head>

<body>



  <div class="grid-container">
    <div class="grid-item">
      <div class="card">
        <img src="{% static "images/admin.png" %}" alt="John" style="width:100%">
        <p class="title">ADMIN</p>
        <p><button><a href="/adminclick">View</a></button></p>
      </div>

    </div>

    <div class="grid-item">
      <div class="card">
        <img src="{% static "images/doctor.png" %}" alt="John" style="width:100%">
        <p class="title">DOCTOR</p>
        <p><button><a href="/doctorclick">View</a></button></p>
      </div>
    </div>

    <div class="grid-item">
      <div class="card">
        <img src="{% static "images/patient.jpg" %}" alt="John" style="width:100%">
        <p class="title">PATIENT</p>
        <p><button><a href="/patientclick">View</a></button></p>
      </div>
    </div>

  </div>
  <!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

</body>

</html>


{% extends 'hospital/admin_base.html' %}
{% load static %}
{% block content %}
<br><br>

<head>
  <meta charset="utf-8">


  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
  <link href="http://netdna.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet">

  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.12.1/css/all.min.css">

  <style type="text/css">
    a:link {
      text-decoration: none;
    }

    .menu {
      top: 50px;
    }

    h6 {
      color: white;
    }

    .order-card {
      color: #fff;
    }

    .bg-c-blue {
      background: linear-gradient(45deg, #4099ff, #73b4ff);
    }

    .bg-c-green {
      background: linear-gradient(45deg, #2ed8b6, #59e0c5);
    }

    .bg-c-yellow {
      background: linear-gradient(45deg, #FFB64D, #ffcb80);
    }

    .bg-c-pink {
      background: linear-gradient(45deg, #FF5370, #ff869a);
    }


    .card {
      border-radius: 5px;
      -webkit-box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      border: none;
      margin-bottom: 30px;
      -webkit-transition: all 0.3s ease-in-out;
      transition: all 0.3s ease-in-out;
    }

    .card .card-block {
      padding: 25px;
    }

    .order-card i {
      font-size: 26px;
    }

    .f-left {
      float: left;
    }

    .f-right {
      float: right;
    }
  </style>
</head>

<link href="https://maxcdn.bootstrapcdn.com/font-awesome/4.3.0/css/font-awesome.min.css" rel="stylesheet">
<div class="container">
  <div class="row">
    <div class="col-md-4 col-xl-3">
      <div class="card bg-c-blue order-card">
        <div class="card-block">
          <a href="/admin-view-patient">
            <h6 class="m-b-20">Patient Record</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-user-injured f-left"></i></h2>

        </div>
      </div>
    </div>

    <div class="col-md-4 col-xl-3">
      <div class="card bg-c-green order-card">
        <div class="card-block">
          <a href="/admin-add-patient">
            <h6 class="m-b-20">Admit Patient</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-user-plus f-left"></i></h2>
        </div>
      </div>
    </div>

    <div class="col-md-4 col-xl-3">
      <div class="card bg-c-yellow order-card">
        <div class="card-block">
          <a href="/admin-approve-patient">
            <h6 class="m-b-20">Approve Patient</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-check-circle f-left"></i></h2>
        </div>
      </div>
    </div>

    <div class="col-md-4 col-xl-3">
      <div class="card bg-c-pink order-card">
        <div class="card-block">
          <a href="/admin-discharge-patient">
            <h6 class="m-b-20">Discharge Patient</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-eject f-left"></i></h2>
        </div>
      </div>
    </div>
  </div>
</div>
<!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

{% endblock content %}



{% extends 'hospital/admin_base.html' %}
{% load widget_tweaks %}
{% block content %}

<head>
  <style media="screen">
    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }

    .menu {
      top: 50px;
    }
  </style>

  <link href="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/js/bootstrap.min.js"></script>
  <script src="//cdnjs.cloudflare.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>
</head>
<br><br>
<!------ update page for doctor by admin(sumit)  ---------->
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  <div class="container register-form">
    <div class="form">
      <div class="note">
        <p>Update Doctor Details</p>
      </div>
      <div class="form-content">
        <div class="row">
          <div class="col-md-6">
            <div class="form-group">
              {% render_field userForm.first_name class="form-control" placeholder="First Name" %}
            </div>
            <div class="form-group">
              {% render_field userForm.username class="form-control" placeholder="Username" %}
            </div>
            <div class="form-group">
              {% render_field doctorForm.mobile class="form-control" placeholder="Mobile" %}
            </div>
            <div class="form-group">
              {% render_field doctorForm.department class="form-control" placeholder="Department" %}
            </div>
          </div>
          <div class="col-md-6">
            <div class="form-group">
              {% render_field userForm.last_name class="form-control" placeholder="Last Name" %}
            </div>
            <div class="form-group">
              {% render_field userForm.password class="form-control" placeholder="Password" %}
            </div>
            <div class="form-group">
              {% render_field doctorForm.address class="form-control" placeholder="Address" %}
            </div>
            <div class="form-group">
              {% render_field doctorForm.profile_pic class="form-control" placeholder="Profile Picture" %}
            </div>
          </div>
        </div>
        <button type="submit" class="btnSubmit">Update</button>
      </div>
    </div>
  </div>
</form>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}



{% extends 'hospital/admin_base.html' %}
{% load widget_tweaks %}
{% block content %}

<head>
  <style media="screen">
    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }

    .menu {
      top: 50px;
    }
  </style>

  <link href="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/js/bootstrap.min.js"></script>
  <script src="//cdnjs.cloudflare.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>
</head>
<br><br>
<!------ update page for doctor by admin(sumit)  ---------->
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  <div class="container register-form">
    <div class="form">
      <div class="note">
        <p>Update Doctor Details</p>
      </div>
      <div class="form-content">
        <div class="row">
          <div class="col-md-6">
            <div class="form-group">
              {% render_field userForm.first_name class="form-control" placeholder="First Name" %}
            </div>
            <div class="form-group">
              {% render_field userForm.username class="form-control" placeholder="Username" %}
            </div>
            <div class="form-group">
              {% render_field patientForm.address class="form-control" placeholder="Address" %}
            </div>
            <div class="form-group">
              {% render_field patientForm.symptoms class="form-control" placeholder="Symptoms" %}
            </div>
            <div class="form-group">
              {% render_field patientForm.profile_pic class="form-control" placeholder="Profile Picture" %}
            </div>
          </div>
          <div class="col-md-6">
            <div class="form-group">
              {% render_field userForm.last_name class="form-control" placeholder="Last Name" %}
            </div>
            <div class="form-group">
              {% render_field userForm.password class="form-control" placeholder="Password" %}
            </div>
            <div class="form-group">
              {% render_field patientForm.mobile class="form-control" placeholder="Mobile" %}
            </div>
            <div class="form-group">
              {% render_field patientForm.assignedDoctorId class="form-control" placeholder="Doctor" %}
            </div>
          </div>
        </div>
        <button type="submit" class="btnSubmit">Update</button>
      </div>
    </div>
  </div>
</form>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}



{% extends 'hospital/admin_base.html' %}
{% block content %}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>

  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Appointments</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Doctor Name</th>
          <th>Patient Name</th>
          <th>Description</th>
          <th>Date</th>
        </tr>
      </thead>
      {% for a in appointments %}
      <tr>
        <td> {{a.doctorName}}</td>
        <td>{{a.patientName}}</td>
        <td>{{a.description}}</td>
        <td>{{a.appointmentDate}}</td>
      </tr>
      {% endfor %}
    </table>
  </div>
</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}



{% extends 'hospital/admin_base.html' %}
{% block content %}
{%load static%}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>

  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Doctors</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>

          <th>Name</th>
          <th>Profile Picture</th>
          <th>Mobile</th>
          <th>Address</th>
          <th>Department</th>
          <th>Update</th>
          <th>Delete</th>
        </tr>
      </thead>
      {% for d in doctors %}
      <tr>

        <td> {{d.get_name}}</td>
        <td> <img src="{% static d.profile_pic.url %}" alt="Profile Pic" height="40px" width="40px" /></td>
        <td>{{d.mobile}}</td>
        <td>{{d.address}}</td>
        <td>{{d.department}}</td>
        <td><a class="btn btn-primary btn-xs" href="{% url 'update-doctor' d.id  %}"><span class="glyphicon glyphicon-edit"></span></a></td>
        <td><a class="btn btn-danger btn-xs" href="{% url 'delete-doctor-from-hospital' d.id  %}"><span class="glyphicon glyphicon-trash"></span></a></td>
      </tr>
      {% endfor %}
    </table>
  </div>
</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}



{% extends 'hospital/admin_base.html' %}
{% block content %}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>

  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Department & Doctors</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Department</th>
          <th>Doctor Name</th>
          <th>Mobile</th>
        </tr>
      </thead>
      {% for d in doctors %}
      <tr>
        <td>{{d.department}}</td>
        <td> {{d.get_name}}</td>
        <td>{{d.mobile}}</td>
      </tr>
      {% endfor %}
    </table>
  </div>
</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}


{% extends 'hospital/admin_base.html' %}
{% block content %}
{%load static%}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>

  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Patient</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Name</th>
          <th>Profile Picture</th>
          <th>Symptoms</th>
          <th>Mobile</th>
          <th>Address</th>
          <th>Update</th>
          <th>Delete</th>
        </tr>
      </thead>
      {% for p in patients %}
      <tr>
        <td> {{p.get_name}}</td>
        <td> <img src="{% static p.profile_pic.url %}" alt="Profile Pic" height="40px" width="40px" /></td>
        <td>{{p.symptoms}}</td>
        <td>{{p.mobile}}</td>
        <td>{{p.address}}</td>
        <td><a class="btn btn-primary btn-xs" href="{% url 'update-patient' p.id  %}"><span class="glyphicon glyphicon-edit"></span></a></td>
        <td><a class="btn btn-danger btn-xs" href="{% url 'delete-patient-from-hospital' p.id  %}"><span class="glyphicon glyphicon-trash"></span></a></td>
      </tr>
      {% endfor %}
    </table>
  </div>
</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}



{% extends 'hospital/homebase.html' %}
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->
{% block content %}

<br>
<br>

<div class="jumbotron" style="margin-bottom:0px;">
  <h1 class="display-4" style="text-align:center;">Hello, Admin</h1>
  <p class="lead">Welcome to Hospital Management System.</p>
  <hr class="my-4">
  <p>You can access various features after Login/SignUp.</p>
  <p class="lead">
    <a class="btn btn-primary btn-lg" href="/adminsignup" role="button">SignUp</a>
    <a class="btn btn-primary btn-lg" href="/adminlogin" role="button">Login</a>
  </p>
</div>

{% endblock content %}


<!DOCTYPE html>
{% load widget_tweaks %}
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  <title>LazyCoder || sumit</title>


  <style type="text/css">
    body {
      color: #aa082e;
      background-color: #b6bde7;
      font-family: 'Roboto', sans-serif;
    }

    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }
  </style>
</head>

<body>
  {% include "hospital/navbar.html" %}
  <br>
  <br>
  <br><br>

  <!--- login page for admin by admin(sumit)  ---------->
  <form method="post">
    {% csrf_token %}
    <div class="container register-form">
      <div class="form">
        <div class="note">
          <p>Admin Login Page</p>
        </div>

        <div class="form-content">
          <div class="row">
            <div class="col-md-6">

              <div class="form-group">
                {% render_field form.username class="form-control" placeholder="Username" %}
              </div>

            </div>
            <div class="col-md-6">

              <div class="form-group">
                {% render_field form.password class="form-control" placeholder="Password" %}
              </div>

            </div>
          </div>
          <button type="submit" class="btnSubmit">Login</button>
          <div class="text-center">Do not have account? <a href="adminsignup">Signup here</a></div>
        </div>
      </div>
    </div>

  </form>

  <br><br><br>
  <!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

  {% include "hospital/footer.html" %}
</body>

</html>


<!DOCTYPE html>

{% load widget_tweaks %}
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  <title>LazyCoder || sumit</title>
  <style type="text/css">
    body {
      color: #aa082e;
      background-color: #b6bde7;
      font-family: 'Roboto', sans-serif;
    }

    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }
  </style>

</head>


<body>
  {% include "hospital/navbar.html" %}
  <br>
  <br>
  <br>
  <br>




  <!--- signup page for admin by admin(sumit)  ---------->
  <form method="post">
    {% csrf_token %}
    <div class="container register-form">
      <div class="form">
        <div class="note">
          <p>Add New Admin To Hospital</p>
        </div>

        <div class="form-content">
          <div class="row">
            <div class="col-md-6">
              <div class="form-group">
                {% render_field form.first_name class="form-control" placeholder="First Name" %}
              </div>
              <div class="form-group">
                {% render_field form.username class="form-control" placeholder="Username" %}
              </div>

            </div>
            <div class="col-md-6">
              <div class="form-group">
                {% render_field form.last_name class="form-control" placeholder="Last Name" %}
              </div>
              <div class="form-group">
                {% render_field form.password class="form-control" placeholder="Password" %}
              </div>

            </div>
          </div>
          <button type="submit" class="btnSubmit">Submit</button>
          <div class="text-center">Already have an account? <a href="adminlogin">Login here</a></div>
        </div>
      </div>
    </div>

  </form>
  <!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

  {% include "hospital/footer.html" %}
</body>

</html>


<!DOCTYPE html>
{% load static %}
<html lang="en" dir="ltr">

<head>
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">

  <title>LazyCoder || sumit</title>


</head>

<body>

  {% include "hospital/navbar.html" %}
  <br><br>

  <center>
    <h3 class='alert alert-success'>Send Us Your Valuable Feedback !</h3>

    <form method="POST">
      <!-- Very Important csrf Token -->
      {% csrf_token %}
      <div class="form-group">
        <p>
        <h3>{{ form.as_p }}</h3>
        </p>
        <br>
        <input type="submit" value="Send Message" class='btn btn-primary btn-lg'>
      </div>
    </form>
  </center>
  {% include "hospital/footer.html" %}
</body>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>


<!DOCTYPE html>
{% load static %}
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">


  <title>LazyCoder || sumit</title>
  <style media="screen">
    .jumbotron {
      margin-bottom: 0px;
    }

    .jumbotron h1 {
      text-align: center;
    }
  </style>

</head>

<body>

  {% include "hospital/navbar.html" %}
  <br><br>
  <div class="jumbotron">
    <h1 class="display-4">Your message sent successfully !</h1>
    <p class="lead">We will respond to your feedback soon</p>
    <hr class="my-4">
    <p>Check other features of website !</p>
    <p class="lead">
      <a class="btn btn-primary btn-lg" href="/" role="button">HOME</a>
    </p>
  </div>

  {% include "hospital/footer.html" %}
</body>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>



{% extends 'hospital/doctor_base.html' %}
{% load static %}
{% block content %}
<br><br>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
  <link href="http://netdna.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet">

    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.12.1/css/all.min.css">

  <style type="text/css">
    a:link {
      text-decoration: none;
    }
    .menu{
      top: 50px;
    }
    h6 {
      color: white;
    }

    .order-card {
      color: #fff;
    }

    .bg-c-blue {
      background: linear-gradient(45deg, #4099ff, #73b4ff);
    }

    .bg-c-green {
      background: linear-gradient(45deg, #2ed8b6, #59e0c5);
    }

    .card {
      border-radius: 5px;
      -webkit-box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      border: none;
      margin-bottom: 30px;
      -webkit-transition: all 0.3s ease-in-out;
      transition: all 0.3s ease-in-out;
    }

    .card .card-block {
      padding: 25px;
    }

    .order-card i {
      font-size: 26px;
    }

    .f-left {
      float: left;
    }

    .f-right {
      float: right;
    }
  </style>
</head>

  <link href="https://maxcdn.bootstrapcdn.com/font-awesome/4.3.0/css/font-awesome.min.css" rel="stylesheet">
  <div class="container">
    <div class="row">
      <div class="col-md-4 col-xl-6">
        <div class="card bg-c-blue order-card">
          <div class="card-block">
            <a href="/doctor-view-appointment">
              <h6 class="m-b-20">View Your Appointment</h6>
            </a>
            <br>
            <h2 class="text-right"><i class="fas fa-calendar f-left"></i></h2>

          </div>
        </div>
      </div>

      <div class="col-md-4 col-xl-6">
        <div class="card bg-c-green order-card">
          <div class="card-block">
            <a href="/doctor-delete-appointment">
              <h6 class="m-b-20">Delete Appointment</h6>
            </a>
            <br>
            <h2 class="text-right"><i class="fas fa-eject f-left"></i></h2>
          </div>
        </div>
      </div>


    </div>
  </div>
  <!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

{% endblock content %}


<!DOCTYPE html>
{% load static %}
<html lang="en">

<head>
  <meta charset="UTF-8">
  <title>LazyCoder || sumit</title>

  <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css">
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/font-awesome/4.7.0/css/font-awesome.min.css">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js"></script>
  <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js"></script>

  <link rel="stylesheet" href="{% static '/style.css' %}">


  <style media="screen">
    a:link {
      text-decoration: none;
    }

    a {
      color: white;
    }

    a:hover {
      color: white;
    }




    /*---------------------------------------
       Social section
    -----------------------------------------*/
    footer {
      padding: 0px 0px 0px 0px;
      background-color: black;
      margin: 0px;
    }


    #ftr {

      padding: 20px;
    }

    .fa {

      font-size: 23px;
      width: 60px;
      text-align: center;
      text-decoration: none;
      margin: 5px 2px;
      border-radius: 50%;
    }

    .fa:hover {
      opacity: 0.5;
      text-decoration: none;
    }

    .fa-facebook {
      background: #3B5998;
      color: white;
      margin-top: 30px;
    }

    .fa-whatsapp {
      background: #25d366;
      color: white;
    }

    .fa-twitter {
      background: #55ACEE;
      color: white;
    }

    .fa-instagram {
      background: #125688;
      color: white;
    }

    p {
      text-align: center;

    }
  </style>
</head>

<body>
  <!-- partial:index.partial.html -->
  <nav class="menu" tabindex="0">
    <div class="smartphone-menu-trigger"></div>
    <header class="avatar">
      <img src="{% static doctor.profile_pic.url %}" alt="Profile Pic" />
      <br><br>
      <h6>Doctor</h6>
      <h2>{{request.user.first_name}}</h2>
    </header>
    <ul>
      <li tabindex="0" class="icon-dashboard"> <a style="color:white; text-decoration:none;" href="/doctor-dashboard"><span>Dashboard</span></a> </li>
      <li tabindex="0" class="icon-users"> <a style="color:white; text-decoration:none;" href="/doctor-patient"><span>Patient</span></a></li>
      <li tabindex="0" class="icon-calendar"> <a style="color:white; text-decoration:none;" href="/doctor-appointment"><span>Appointments</span></a></li>
    </ul>
  </nav>
  <main>
    <!-- nav start -->
    <div class="bs-example">
      <nav class="navbar navbar-expand-md  navbar-dark fixed-top" style="background:#337AB7;">
        <a href="/admin-dashboard" class="navbar-brand">HOSPITAL MANAGEMENT</a>
        <button type="button" class="navbar-toggler" data-toggle="collapse" data-target="#navbarCollapse">
          <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse justify-content-between" id="navbarCollapse">
          <div class="navbar-nav" style=" margin-left: 90%;">

            <a href="/logout" class="nav-item nav-link">Logout</a>
          </div>
        </div>
      </nav>
    </div>
    <!-- nav end -->
    <br><br>
    <!-- content start-->
    {% block content %}
    {% endblock content %}
    <!-- content end-->
    <br><br><br><br><br><br><br>
    <footer>
      <p>
        <a id="ftr" href="https://facebook.com/sumit.luv/" class="fa fa-facebook"></a>
        <a id="ftr" href="https://api.whatsapp.com/send?phone=919572181024&text=Hello%20Sumit.%0d%0aHow%20are%20you%20%3f%0d%0aI%20came%20from%20your%20website.&source=&data=#" class="fa fa-whatsapp"></a>
        <a id="ftr" href="https://instagram.com/sumit.luv" class="fa fa-instagram"></a>
        <a id="ftr" href="https://twitter.com/sumitkumar1503" class="fa fa-twitter"></a>
      </p>
      <br>
      <div class="container">
        <div class="row">
          <div class="col-md-12 col-sm-12">
            <div style="color:#ffffff;" class="wow fadeInUp footer-copyright">
              <p>Made in India <br>
                Copyright &copy; 2020 LazyCoder </p>
            </div>
          </div>
        </div>
      </div>
    </footer>
  </main>
  <!-- partial -->
</body>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>


{% extends 'hospital/doctor_base.html' %}
{% load static %}
{% block content %}
{%include 'hospital/doctor_dashboard_cards.html'%}
<br><br><br><br>
<div class="container">
  <div class="row">
    <div class="panel panel-primary" style="margin-left:15%;">
      <div class="panel-heading" style="text-align:center;">
        <h6 class="panel-title">Recent Appointments For You</h6>
      </div>
      <table class="table table-hover" id="dev-table">
        <thead>
          <tr>
            <th>Patient Name</th>
            <th>Picture</th>
            <th>Description</th>
            <th>Mobile</th>
            <th>Address</th>
            <th>Date</th>
          </tr>
        </thead>
        {% for a,p in appointments %}
        <tr>
          <td>{{a.patientName}}</td>
          <td> <img src="{% static p.profile_pic.url %}" alt="Profile Pic" height="40px" width="40px" /></td>
          <td>{{a.description}}</td>
          <td>{{p.mobile}}</td>
          <td>{{p.address}}</td>
          <td>{{a.appointmentDate}}</td>
        </tr>
        {% endfor %}
      </table>
    </div>
  </div>
</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}


<!DOCTYPE html>
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <title></title>

  <style media="screen">
    .market-update-block {
      padding: 2em 2em;
      background: #999;
    }

    .market-update-block h3 {
      color: #fff;
      font-size: 2.5em;
      font-family: 'Carrois Gothic', sans-serif;
    }

    .market-update-block h4 {
      font-size: 1.2em;
      color: #fff;
      margin: 0.3em 0em;
      font-family: 'Carrois Gothic', sans-serif;
    }

    .market-update-block p {
      color: #fff;
      font-size: 0.8em;
      line-height: 1.8em;
    }

    .market-update-block.clr-block-1 {
      background: #ff0000;
      margin-right: 0.8em;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-2 {
      background: #FC8213;
      margin-right: 0.8em;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-3 {
      background: #1355f9;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-1:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-2:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-3:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-right i.fa.fa-user-o {
      font-size: 3em;
      color: #68AE00;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-right i.fa.fa-check-circle-o {
      font-size: 3em;
      color: #FC8213;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-right i.fa.fa-calendar {
      font-size: 3em;
      color: #337AB7;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-left {
      padding: 0px;
    }
  </style>
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
  <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/js/bootstrap.min.js"></script>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">
</head>

<body>
  <div class="market-updates">
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-1">
        <div class="col-md-8 market-update-left">
          <h3>{{appointmentcount}}</h3>
          <h4>Appointments For You</h4>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-calendar"> </i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-2">
        <div class="col-md-8 market-update-left">
          <h3>{{patientcount}}</h3>
          <h4>Patient Under You</h4>

        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-user-o"></i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-3">
        <div class="col-md-8 market-update-left">
          <h3>{{patientdischarged}}</h3>
          <h4>Your Patient Discharged</h4>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-check-circle-o" aria-hidden="true"></i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="clearfix"> </div>
  </div>
</body>
<!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

</html>



<!DOCTYPE html>
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <title></title>

  <style media="screen">
    .market-update-block {
      padding: 2em 2em;
      background: #999;
    }

    .market-update-block h3 {
      color: #fff;
      font-size: 2.5em;
      font-family: 'Carrois Gothic', sans-serif;
    }

    .market-update-block h4 {
      font-size: 1.2em;
      color: #fff;
      margin: 0.3em 0em;
      font-family: 'Carrois Gothic', sans-serif;
    }

    .market-update-block p {
      color: #fff;
      font-size: 0.8em;
      line-height: 1.8em;
    }

    .market-update-block.clr-block-1 {
      background: #ff0000;
      margin-right: 0.8em;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-2 {
      background: #FC8213;
      margin-right: 0.8em;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-3 {
      background: #1355f9;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-1:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-2:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-3:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-right i.fa.fa-user-o {
      font-size: 3em;
      color: #68AE00;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-right i.fa.fa-check-circle-o {
      font-size: 3em;
      color: #FC8213;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-right i.fa.fa-calendar {
      font-size: 3em;
      color: #337AB7;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-left {
      padding: 0px;
    }
  </style>
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
  <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/js/bootstrap.min.js"></script>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">
</head>

<body>
  <div class="market-updates">
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-1">
        <div class="col-md-8 market-update-left">
          <h3>{{appointmentcount}}</h3>
          <h4>Appointments For You</h4>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-calendar"> </i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-2">
        <div class="col-md-8 market-update-left">
          <h3>{{patientcount}}</h3>
          <h4>Patient Under You</h4>

        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-user-o"></i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-3">
        <div class="col-md-8 market-update-left">
          <h3>{{patientdischarged}}</h3>
          <h4>Your Patient Discharged</h4>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-check-circle-o" aria-hidden="true"></i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="clearfix"> </div>
  </div>
</body>
<!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

</html>


{% extends 'hospital/doctor_base.html' %}
{% block content %}
{%load static%}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>

  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Delete Your Appointments</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Patient Name</th>
          <th>Picture</th>
          <th>Description</th>
          <th>Delete</th>
        </tr>
      </thead>
      {% for a,p in appointments %}
      <tr>
        <td>{{a.patientName}}</td>
        <td> <img src="{% static p.profile_pic.url %}" alt="Profile Pic" height="40px" width="40px" /></td>
        <td>{{a.description}}</td>
        <td><a class="btn btn-danger btn-xs" href="{% url 'delete-appointment' a.id  %}"><span class="glyphicon glyphicon-trash"></span></a></td>
      </tr>
      {% endfor %}
    </table>
  </div>
</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}


{% extends 'hospital/doctor_base.html' %}
{% load static %}
{% block content %}
<br><br>


<head>
  <meta charset="utf-8">


  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
  <link href="http://netdna.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet">

  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.12.1/css/all.min.css">

  <style type="text/css">
    a:link {
      text-decoration: none;
    }

    .menu {
      top: 50px;
    }

    h6 {
      color: white;
    }

    .order-card {
      color: #fff;
    }

    .bg-c-blue {
      background: linear-gradient(45deg, #4099ff, #73b4ff);
    }

    .bg-c-green {
      background: linear-gradient(45deg, #2ed8b6, #59e0c5);
    }

    .bg-c-yellow {
      background: linear-gradient(45deg, #FFB64D, #ffcb80);
    }

    .bg-c-pink {
      background: linear-gradient(45deg, #FF5370, #ff869a);
    }


    .card {
      border-radius: 5px;
      -webkit-box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      border: none;
      margin-bottom: 30px;
      -webkit-transition: all 0.3s ease-in-out;
      transition: all 0.3s ease-in-out;
    }

    .card .card-block {
      padding: 25px;
    }

    .order-card i {
      font-size: 26px;
    }

    .f-left {
      float: left;
    }

    .f-right {
      float: right;
    }
  </style>
</head>

<link href="https://maxcdn.bootstrapcdn.com/font-awesome/4.3.0/css/font-awesome.min.css" rel="stylesheet">
<div class="container">
  <div class="row">
    <div class="col-md-4 col-xl-6">
      <div class="card bg-c-blue order-card">
        <div class="card-block">
          <a href="/doctor-view-patient">
            <h6 class="m-b-20">Your Patient Record</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-user-injured f-left"></i></h2>

        </div>
      </div>
    </div>


    <div class="col-md-4 col-xl-6">
      <div class="card bg-c-pink order-card">
        <div class="card-block">
          <a href="/doctor-view-discharge-patient">
            <h6 class="m-b-20">Your Discharged Patient</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-eject f-left"></i></h2>
        </div>
      </div>
    </div>
  </div>
</div>
<!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

{% endblock content %}




{% extends 'hospital/doctor_base.html' %}
{% load static %}
{% block content %}
<br><br>


<head>
  <meta charset="utf-8">


  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
  <link href="http://netdna.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet">

  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.12.1/css/all.min.css">

  <style type="text/css">
    a:link {
      text-decoration: none;
    }

    .menu {
      top: 50px;
    }

    h6 {
      color: white;
    }

    .order-card {
      color: #fff;
    }

    .bg-c-blue {
      background: linear-gradient(45deg, #4099ff, #73b4ff);
    }

    .bg-c-green {
      background: linear-gradient(45deg, #2ed8b6, #59e0c5);
    }

    .bg-c-yellow {
      background: linear-gradient(45deg, #FFB64D, #ffcb80);
    }

    .bg-c-pink {
      background: linear-gradient(45deg, #FF5370, #ff869a);
    }


    .card {
      border-radius: 5px;
      -webkit-box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      border: none;
      margin-bottom: 30px;
      -webkit-transition: all 0.3s ease-in-out;
      transition: all 0.3s ease-in-out;
    }

    .card .card-block {
      padding: 25px;
    }

    .order-card i {
      font-size: 26px;
    }

    .f-left {
      float: left;
    }

    .f-right {
      float: right;
    }
  </style>
</head>

<link href="https://maxcdn.bootstrapcdn.com/font-awesome/4.3.0/css/font-awesome.min.css" rel="stylesheet">
<div class="container">
  <div class="row">
    <div class="col-md-4 col-xl-6">
      <div class="card bg-c-blue order-card">
        <div class="card-block">
          <a href="/doctor-view-patient">
            <h6 class="m-b-20">Your Patient Record</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-user-injured f-left"></i></h2>

        </div>
      </div>
    </div>


    <div class="col-md-4 col-xl-6">
      <div class="card bg-c-pink order-card">
        <div class="card-block">
          <a href="/doctor-view-discharge-patient">
            <h6 class="m-b-20">Your Discharged Patient</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-eject f-left"></i></h2>
        </div>
      </div>
    </div>
  </div>
</div>
<!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

{% endblock content %}




{% extends 'hospital/doctor_base.html' %}
{% block content %}
{%load static%}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>

  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Your Appointments</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Patient Name</th>
          <th>Picture</th>
          <th>Description</th>
          <th>Mobile</th>
          <th>Address</th>
          <th>Appointment Date</th>
        </tr>
      </thead>
      {% for a,p in appointments %}
      <tr>
        <td>{{a.patientName}}</td>
        <td> <img src="{% static p.profile_pic.url %}" alt="Profile Pic" height="40px" width="40px" /></td>
        <td>{{a.description}}</td>
        <td>{{p.mobile}}</td>
        <td>{{p.address}}</td>
        <td>{{a.appointmentDate}}</td>
      </tr>
      {% endfor %}
    </table>
  </div>
</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}


{% extends 'hospital/doctor_base.html' %}
{% block content %}
{%load static%}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>

  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Your Discharged Patient List</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Name</th>

          <th>Admit Date</th>
          <th>Release Date</th>
          <th>Symptoms</th>
          <th>Mobile</th>
          <th>Address</th>

        </tr>
      </thead>
      {% for p in dischargedpatients %}
      <tr>
        <td> {{p.patientName}}</td>
        <td>{{p.admitDate}}</td>
        <td>{{p.releaseDate}}</td>
        <td>{{p.symptoms}}</td>
        <td>{{p.mobile}}</td>
        <td>{{p.address}}</td>
      </tr>
      {% endfor %}
    </table>
  </div>
</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}

{% extends 'hospital/doctor_base.html' %}
{% block content %}
{%load static%}

<head>
  <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  <script src="//code.jquery.com/jquery-1.11.1.min.js"></script>

  <style media="screen">
    a:link {
      text-decoration: none;
    }

    h6 {
      text-align: center;
    }

    .row {
      margin: 100px;
    }
  </style>
</head>
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->
<div class="container">
  <div class="panel panel-primary">
    <div class="panel-heading">
      <h6 class="panel-title">Your Total Patient List</h6>
    </div>
    <table class="table table-hover" id="dev-table">
      <thead>
        <tr>
          <th>Name</th>
          <th>Profile Picture</th>
          <th>Symptoms</th>
          <th>Mobile</th>
          <th>Address</th>

        </tr>
      </thead>
      {% for p in patients %}
      <tr>
        <td> {{p.get_name}}</td>
        <td> <img src="{% static p.profile_pic.url %}" alt="Profile Pic" height="40px" width="40px" /></td>
        <td>{{p.symptoms}}</td>
        <td>{{p.mobile}}</td>
        <td>{{p.address}}</td>
      </tr>
      {% endfor %}
    </table>
  </div>
</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}


<!DOCTYPE html>

<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  <title>LazyCoder || sumit</title>


  <style media="screen">
    .jumbotron {
      margin-top: 0px;
      margin-bottom: 0px;
    }

    .jumbotron h1 {
      text-align: center;
    }

    .alert {
      margin: 0px;
    }
  </style>

</head>

<body>
  {% include "hospital/navbar.html" %}
  <br>
  <br>

  <div class="jumbotron" style="margin-top: 0px;
    margin-bottom: 0px;">
    <h1 class="display-4">Hello {{request.user.first_name}}</h1>
    <p class="lead">Your Account is not approved till now <br><br>Our Team is checking your profile <br><br> Soon your Account will be confirmed !!!</p>
    <hr class="my-4">
    <p>Check Later</p>
    <p class="lead">
      <a class="btn btn-primary btn-lg" href="/logout" role="button">Logout For Now</a>
    </p>
  </div>

  {% include "hospital/footer.html" %}
</body>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>

{% extends 'hospital/homebase.html' %}
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% block content %}

<br>
<br>

<div class="jumbotron" style="margin-bottom:0px;">
  <h1 class="display-4" style="text-align:center;">Hello, Doctor</h1>
  <p class="lead">Welcome to Hospital Management System.</p>
  <hr class="my-4">
  <p>You can access various features after Login/SignUp.</p>
  <p class="lead">
    <a class="btn btn-primary btn-lg" href="/doctorsignup" role="button">Apply</a>
    <a class="btn btn-primary btn-lg" href="/doctorlogin" role="button">Login</a>
  </p>
</div>

{% endblock content %}

<!DOCTYPE html>
{% load widget_tweaks %}
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  <title>LazyCoder || sumit</title>


  <style type="text/css">
    body {
      color: #aa082e;
      background-color: #b6bde7;
      font-family: 'Roboto', sans-serif;
    }

    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }
  </style>

</head>

<body>
  {% include "hospital/navbar.html" %}
  <br>
  <br>
  <br><br>

  <!--- login page for doctor by admin(sumit)  ---------->
  <form method="post">
    {% csrf_token %}
    <div class="container register-form">
      <div class="form">
        <div class="note">
          <p>Doctor Login Page</p>
        </div>

        <div class="form-content">
          <div class="row">
            <div class="col-md-6">

              <div class="form-group">
                {% render_field form.username class="form-control" placeholder="Username" %}
              </div>

            </div>
            <div class="col-md-6">

              <div class="form-group">
                {% render_field form.password class="form-control" placeholder="Password" %}
              </div>

            </div>
          </div>
          <button type="submit" class="btnSubmit">Login</button>
          <div class="text-center">Do not have account? <a href="doctorsignup">Signup here</a></div>
        </div>
      </div>
    </div>

  </form>

  <br><br><br>
  <!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

  {% include "hospital/footer.html" %}
</body>

</html>

<!DOCTYPE html>

{% load widget_tweaks %}
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  <title>LazyCoder || sumit</title>
  <style type="text/css">
    body {
      color: #aa082e;
      background-color: #b6bde7;
      font-family: 'Roboto', sans-serif;
    }

    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }
  </style>

</head>


<body>
  {% include "hospital/navbar.html" %}
  <br>
  <br>
  <br>
  <br>




  <!--- signup page for doctor by admin(sumit)  ---------->
  <form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    <div class="container register-form">
      <div class="form">
        <div class="note">
          <p>Register In Hospital</p>
        </div>

        <div class="form-content">
          <div class="row">
            <div class="col-md-6">
              <div class="form-group">
                {% render_field userForm.first_name class="form-control" placeholder="First Name" %}
              </div>
              <div class="form-group">
                {% render_field userForm.username class="form-control" placeholder="Username" %}
              </div>
              <div class="form-group">
                {% render_field doctorForm.department class="form-control" placeholder="Department" %}
              </div>
              <div class="form-group">
                {% render_field doctorForm.address class="form-control" placeholder="Address" %}
              </div>

            </div>
            <div class="col-md-6">
              <div class="form-group">
                {% render_field userForm.last_name class="form-control" placeholder="Last Name" %}
              </div>
              <div class="form-group">
                {% render_field userForm.password class="form-control" placeholder="Password" %}
              </div>
              <div class="form-group">
                {% render_field doctorForm.mobile class="form-control"  placeholder="Mobile" %}
              </div>
              <div class="form-group">
                {% render_field doctorForm.profile_pic required="required" class="form-control" placeholder="Profile Picture" %}
              </div>

            </div>
          </div>
          <button type="submit" class="btnSubmit">Register</button>
          <div class="text-center">Already have an account? <a href="doctorlogin">Login here</a></div>
        </div>
      </div>
    </div>

  </form>

  {% include "hospital/footer.html" %}
</body>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>

<!DOCTYPE html>
<html lang="en" dir="ltr">


<head>
  <meta charset="utf-8">
  <style>
    @page {
      size: A4;
      margin: 1cm;

      @frame footer {
        -pdf-frame-content: footerContent;
        bottom: 0cm;
        margin-left: 9cm;
        margin-right: 9cm;
        height: 1cm;
      }
    }

    .invoice-box {
      max-width: 800px;
      margin: auto;
      padding: 30px;
      border: 1px solid #eee;
      box-shadow: 0 0 10px rgba(0, 0, 0, .15);
      font-size: 16px;
      line-height: 24px;
      font-family: 'Helvetica Neue', 'Helvetica', Helvetica, Arial, sans-serif;
      color: #555;
    }

    .invoice-box table {
      width: 100%;
      line-height: inherit;
      text-align: left;
    }

    .invoice-box table td {
      padding: 5px;
      vertical-align: top;
    }

    .invoice-box table tr td:nth-child(2) {
      text-align: right;
    }

    .invoice-box table tr.top table td {
      padding-bottom: 20px;
    }

    .invoice-box table tr.top table td.title {
      font-size: 45px;
      line-height: 45px;
      color: #333;
    }

    .invoice-box table tr.information table td {
      padding-bottom: 40px;
    }

    .invoice-box table tr.heading td {
      background: #eee;
      border-bottom: 1px solid #ddd;
      font-weight: bold;
    }

    .invoice-box table tr.details td {
      padding-bottom: 20px;
    }

    .invoice-box table tr.item td {
      border-bottom: 1px solid #eee;
    }

    .invoice-box table tr.item.last td {
      border-bottom: none;
    }

    .invoice-box table tr.total td:nth-child(2) {
      border-top: 2px solid #eee;
      font-weight: bold;
    }

    @media only screen and (max-width: 600px) {
      .invoice-box table tr.top table td {
        width: 100%;
        display: block;
        text-align: center;
      }

      .invoice-box table tr.information table td {
        width: 100%;
        display: block;
        text-align: center;
      }
    }

    /** RTL **/
    .rtl {
      direction: rtl;
      font-family: Tahoma, 'Helvetica Neue', 'Helvetica', Helvetica, Arial, sans-serif;
    }

    .rtl table {
      text-align: right;
    }

    .rtl table tr td:nth-child(2) {
      text-align: left;
    }
  </style>
</head>

<body>

  <br><br><br>
  <div class="invoice-box">
    <table cellpadding="0" cellspacing="0">
      <tr class="top">
        <td colspan="2">
          <table>
            <tr>
              <td class="title">
                <h5>Hospital Management</h5>
              </td>

              <td>

                Admit Date: {{admitDate}}<br>
                Release Date: {{releaseDate}}<br>
                Days Spent: {{daySpent}}
              </td>
            </tr>
          </table>
        </td>
      </tr>

      <tr class="information">
        <td colspan="2">
          <table>
            <tr>
              <td>
                Patient Name : {{patientName}}<br>
                Patient Mobile : {{mobile}}<br>
                Patient Addres : {{address}}<br>
              </td>

              <td>
                Doctor Name :<br>
                {{assignedDoctorName}}<br>

              </td>
            </tr>
          </table>
        </td>
      </tr>


      <tr class="information">
        <td colspan="2">
          <table>
            <tr>
              <td>
                Disease and Symptoms :<br>
                &nbsp &nbsp &nbsp &nbsp &nbsp {{symptoms}}
              </td>

            </tr>
          </table>
        </td>
      </tr>



      <tr class="information">
        <td colspan="2">
          <table>
            <tr>
              <td>
                Charges :<br><br>
                Room Charge of {{daySpent}} Days : {{roomCharge}}<br>
                Doctor Fee : {{doctorFee}}<br>
                Medicine Cost : {{medicineCost}}<br>
                Other Charge : {{OtherCharge}} <br><br>
                &nbsp &nbsp &nbsp &nbsp &nbsp Total Rupees : {{total}}
              </td>
            </tr>
          </table>
        </td>
      </tr>


    </table>
  </div>
</body>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>
<!DOCTYPE html>
<html>

<head>


  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/font-awesome/4.7.0/css/font-awesome.min.css">


  <style>
    /*---------------------------------------
   Social section
-----------------------------------------*/
    footer {
      padding: 0px 0px 0px 0px;
      background-color: black;
      margin: 0px;
    }

    .fa {
      padding: 20px;
      font-size: 23px;
      width: 60px;
      text-align: center;
      text-decoration: none;
      margin: 5px 2px;
      border-radius: 50%;
    }

    .fa:hover {
      opacity: 0.5;
      text-decoration: none;
    }

    .fa-facebook {
      background: #3B5998;
      color: white;
      margin-top: 30px;
    }

    .fa-whatsapp {
      background: #25d366;
      color: white;
    }

    .fa-twitter {
      background: #55ACEE;
      color: white;
    }

    .fa-instagram {
      background: #125688;
      color: white;
    }

    p {
      text-align: center;

    }
  </style>
</head>

<footer>

  <p>
    <a href="https://facebook.com/sumit.luv/" class="fa fa-facebook"></a>
    <a href="https://api.whatsapp.com/send?phone=919572181024&text=Hello%20Sumit.%0d%0aHow%20are%20you%20%3f%0d%0aI%20came%20from%20your%20website.&source=&data=#" class="fa fa-whatsapp"></a>
    <a href="https://instagram.com/sumit.luv" class="fa fa-instagram"></a>
    <a href="https://twitter.com/sumitkumar1503" class="fa fa-twitter"></a>
  </p>

  <br>
  <div class="container">
    <div class="row">
      <div class="col-md-12 col-sm-12">
        <div style="color:#ffffff;" class="wow fadeInUp footer-copyright">
          <p>Made in India <br>
            Copyright &copy; 2020 LazyCoder </p>
        </div>
      </div>
    </div>
  </div>
</footer>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>
<!DOCTYPE html>
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <title>LazyCoder || sumit</title>

</head>

<body>
  {% include "hospital/navbar.html" %}
  {%block content%}

  {%endblock content%}
  {% include "hospital/footer.html" %}
</body>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>
{% extends 'hospital/homebase.html' %}
{% load static %}
<!--
written By : sumit kumar
facebook : fb.com/sumit.luv
-->


{% block content %}


<head>
  <style media="screen">
    .jumbotron {
      margin-top: 0px;
      margin-bottom: 0px;
      background-image: url('{% static "images/bg.jpg" %}');
      background-size: cover;
      background-repeat: no-repeat;
    }

    .jumbotron h5,
    h3 {
      text-align: center;
    }


    .alert {
      margin: 0px;
    }



    .glow {
      font-size: 50px;
      color: white;
      text-align: center;
      -webkit-animation: glow 1s ease-in-out infinite alternate;
      -moz-animation: glow 1s ease-in-out infinite alternate;
      animation: glow 1s ease-in-out infinite alternate;
    }

    @-webkit-keyframes glow {
      from {
        text-shadow: 0 0 10px #eeeeee, 0 0 20px #000000, 0 0 30px #000000, 0 0 40px #000000, 0 0 50px #9554b3, 0 0 60px #9554b3, 0 0 70px #9554b3;
      }

      to {
        text-shadow: 0 0 20px #eeeeee, 0 0 30px #ff4da6, 0 0 40px #ff4da6, 0 0 50px #ff4da6, 0 0 60px #ff4da6, 0 0 70px #ff4da6, 0 0 80px #ff4da6;
      }
    }
  </style>
</head>
<br>
<br>

<div class="jumbotron" style="margin-bottom: 0px;margin-top: 0px;">
  <br>
  <h5 class="display-3 glow">You’ll Love the Way We Care for You</h5>
  <br><br><br><br><br>
  <p>
  <h3>Emergency ?</h3>

  <p class="lead">
    <a class="btn btn-primary btn-lg" href="/patientclick" role="button">Take Appointment</a>
  </p>
  <br><br>
</div>

<br><br><br><br>

{% include "hospital/admin_doctor_patient_card.html" %}
<br><br><br>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}
<!DOCTYPE html>
{% load static %}
<html lang="en">

<head>
  <meta charset="utf-8">
  <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css">
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/font-awesome/4.7.0/css/font-awesome.min.css">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js"></script>
  <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js"></script>

  <style type="text/css">
    .bs-example {
      margin: 0px;

    }

    .navbar-brand {
      font-size: 20px;
      font-family: sans-serif;

    }
  </style>
</head>

<body>



  <div class="bs-example">
    <nav class="navbar navbar-expand-md navbar-dark fixed-top" style="background:#337AB7;">
      <a href="/" class="navbar-brand">Hospital Management</a>
      <button type="button" class="navbar-toggler" data-toggle="collapse" data-target="#navbarCollapse">
        <span class="navbar-toggler-icon"></span>
      </button>

      <div class="collapse navbar-collapse justify-content-between" id="navbarCollapse">
        <div class="navbar-nav">

          <a href="/adminclick" class="nav-item nav-link">Admin</a>
          <a href="/doctorclick" class="nav-item nav-link">Doctor</a>
          <a href="/patientclick" class="nav-item nav-link">Patient</a>


        </div>

        <div class="navbar-nav">
          <a href="/aboutus" class="nav-item nav-link">About Us</a>
          <a href="/contactus" class="nav-item nav-link">Contact Us</a>
        </div>

      </div>
    </nav>
  </div>

  <!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

</body>

</html>


{% extends 'hospital/patient_base.html' %}
{% load static %}
{% block content %}
<br><br>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
  <link href="http://netdna.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" rel="stylesheet">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.12.1/css/all.min.css">
  <style type="text/css">
    a:link {
      text-decoration: none;
    }

    .menu {
      top: 50px;
    }

    h6 {
      color: white;
    }

    .order-card {
      color: #fff;
    }

    .bg-c-blue {
      background: linear-gradient(45deg, #4099ff, #73b4ff);
    }

    .bg-c-green {
      background: linear-gradient(45deg, #2ed8b6, #59e0c5);
    }

    .bg-c-yellow {
      background: linear-gradient(45deg, #FFB64D, #ffcb80);
    }

    .bg-c-pink {
      background: linear-gradient(45deg, #FF5370, #ff869a);
    }


    .card {
      border-radius: 5px;
      -webkit-box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      box-shadow: 0 1px 2.94px 0.06px rgba(4, 26, 55, 0.16);
      border: none;
      margin-bottom: 30px;
      -webkit-transition: all 0.3s ease-in-out;
      transition: all 0.3s ease-in-out;
    }

    .card .card-block {
      padding: 25px;
    }

    .order-card i {
      font-size: 26px;
    }

    .f-left {
      float: left;
    }

    .f-right {
      float: right;
    }
  </style>
</head>

<link href="https://maxcdn.bootstrapcdn.com/font-awesome/4.3.0/css/font-awesome.min.css" rel="stylesheet">
<div class="container">
  <div class="row">
    <div class="col-md-4 col-xl-6">
      <div class="card bg-c-blue order-card">
        <div class="card-block">
          <a href="/patient-view-appointment">
            <h6 class="m-b-20">View Your Appointment</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-calendar f-left"></i></h2>

        </div>
      </div>
    </div>

    <div class="col-md-4 col-xl-6">
      <div class="card bg-c-green order-card">
        <div class="card-block">
          <a href="/patient-book-appointment">
            <h6 class="m-b-20">Book Appointment</h6>
          </a>
          <br>
          <h2 class="text-right"><i class="fas fa-book-medical f-left"></i></h2>
        </div>
      </div>
    </div>
  </div>
</div>
<!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

{% endblock content %}

<!DOCTYPE html>
{% load static %}
<html lang="en">

<head>
  <meta charset="UTF-8">
  <title>LazyCoder || sumit</title>

  <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css">
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/font-awesome/4.7.0/css/font-awesome.min.css">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js"></script>
  <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js"></script>

  <link rel="stylesheet" href="{% static '/style.css' %}">


  <style media="screen">
    a:link {
      text-decoration: none;
    }

    a {
      color: white;
    }

    a:hover {
      color: white;
    }




    /*---------------------------------------
       Social section
    -----------------------------------------*/
    footer {
      padding: 0px 0px 0px 0px;
      background-color: black;
      margin: 0px;
    }


    #ftr {

      padding: 20px;
    }

    .fa {

      font-size: 23px;
      width: 60px;
      text-align: center;
      text-decoration: none;
      margin: 5px 2px;
      border-radius: 50%;
    }

    .fa:hover {
      opacity: 0.5;
      text-decoration: none;
    }

    .fa-facebook {
      background: #3B5998;
      color: white;
      margin-top: 30px;
    }

    .fa-whatsapp {
      background: #25d366;
      color: white;
    }

    .fa-twitter {
      background: #55ACEE;
      color: white;
    }

    .fa-instagram {
      background: #125688;
      color: white;
    }

    p {
      text-align: center;

    }
  </style>
</head>

<body>
  <!-- partial:index.partial.html -->
  <nav class="menu" tabindex="0">
    <div class="smartphone-menu-trigger"></div>
    <header class="avatar">
      <img src="{% static patient.profile_pic.url %}" alt="Profile Pic" />
      <br><br>
      <h6>Patient</h6>
      <h2>{{request.user.first_name}}</h2>
    </header>
    <ul>
      <li tabindex="0" class="icon-dashboard"> <a style="color:white; text-decoration:none;" href="/patient-dashboard"><span>Dashboard</span></a> </li>
      <li tabindex="0" class="icon-calendar"> <a style="color:white; text-decoration:none;" href="/patient-appointment"><span>Appointments</span></a></li>
      <li tabindex="0" class="icon-users"> <a style="color:white; text-decoration:none;" href="/patient-discharge"><span>Discharge</span></a></li>
    </ul>
  </nav>
  <main>
    <!-- nav start -->
    <div class="bs-example">
      <nav class="navbar navbar-expand-md  navbar-dark fixed-top" style="background:#337AB7;">
        <a href="/admin-dashboard" class="navbar-brand">HOSPITAL MANAGEMENT</a>
        <button type="button" class="navbar-toggler" data-toggle="collapse" data-target="#navbarCollapse">
          <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse justify-content-between" id="navbarCollapse">
          <div class="navbar-nav" style=" margin-left: 90%;">

            <a href="/logout" class="nav-item nav-link">Logout</a>
          </div>
        </div>
      </nav>
    </div>
    <!-- nav end -->
    <br><br>
    <!-- content start-->
    {% block content %}
    {% endblock content %}
    <!-- content end-->
    <br><br><br><br><br><br><br>
    <footer>
      <p>
        <a id="ftr" href="https://facebook.com/sumit.luv/" class="fa fa-facebook"></a>
        <a id="ftr" href="https://api.whatsapp.com/send?phone=919572181024&text=Hello%20Sumit.%0d%0aHow%20are%20you%20%3f%0d%0aI%20came%20from%20your%20website.&source=&data=#" class="fa fa-whatsapp"></a>
        <a id="ftr" href="https://instagram.com/sumit.luv" class="fa fa-instagram"></a>
        <a id="ftr" href="https://twitter.com/sumitkumar1503" class="fa fa-twitter"></a>
      </p>
      <br>
      <div class="container">
        <div class="row">
          <div class="col-md-12 col-sm-12">
            <div style="color:#ffffff;" class="wow fadeInUp footer-copyright">
              <p>Made in India <br>
                Copyright &copy; 2020 LazyCoder </p>
            </div>
          </div>
        </div>
      </div>
    </footer>
  </main>
  <!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

</body>

</html>
{% extends 'hospital/patient_base.html' %}
{% load widget_tweaks %}
{% block content %}

<head>
  <style media="screen">
    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }

    .menu {
      top: 50px;
    }
  </style>

  <link href="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/css/bootstrap.min.css" rel="stylesheet" id="bootstrap-css">
  <script src="//maxcdn.bootstrapcdn.com/bootstrap/4.1.1/js/bootstrap.min.js"></script>
  <script src="//cdnjs.cloudflare.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>
</head>
<br><br>
<!------ add appointment page by patient(sumit)  ---------->
<form method="post">
  {% csrf_token %}
  <div class="container register-form">
    <div class="form">
      <div class="note">
        <p>Book Appointment Details</p>
      </div>
      <div class="form-content">
        <div class="row">
          <div class="col-md-12">
            <div class="form-group">
              {% render_field appointmentForm.description class="form-control" placeholder="Description" %}
            </div>
            <div class="form-group">
              {% render_field appointmentForm.doctorId class="form-control" placeholder="doctor" %}
            </div>
            



          </div>

        </div>
        <button type="submit" class="btnSubmit">Book</button>
      </div>
    </div>
  </div>
</form>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->


{% endblock content %}


{% extends 'hospital/patient_base.html' %}
{% load static %}
{% block content %}
{%include 'hospital/patient_dashboard_cards.html'%}
<br><br><br><br>

<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}
<!DOCTYPE html>
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <title></title>

  <style media="screen">
    .market-update-block {
      padding: 2em 2em;
      background: #999;
    }

    .market-update-block h3 {
      color: #fff;
      font-size: 1.5em;
      font-family: 'Carrois Gothic', sans-serif;
    }

    .market-update-block h4 {
      font-size: 1.2em;
      color: #fff;
      margin: 0.3em 0em;
      font-family: 'Carrois Gothic', sans-serif;
    }

    .market-update-block p {
      color: #fff;
      font-size: 0.8em;
      line-height: 1.8em;
    }

    .market-update-block.clr-block-1 {
      background: #ff0000;
      margin-right: 0.8em;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-2 {
      background: #4f5905;
      margin-right: 0.8em;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-3 {
      background: #1355f9;
      box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-1:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-2:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-block.clr-block-3:hover {
      background: #3C3C3C;
      transition: 0.5s all;
      -webkit-transition: 0.5s all;
      -moz-transition: 0.5s all;
      -o-transition: 0.5s all;
    }

    .market-update-right i.fa.fa-user-o,
    i.fa.fa-map-marker {
      font-size: 3em;
      color: #68AE00;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-right i.fa.fa-info-circle,
    i.fa.fa-list-alt {
      font-size: 3em;
      color: #FC8213;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }

    .market-update-right i.fa.fa-mobile,
    i.fa.fa-calendar-o {
      font-size: 3em;
      color: #337AB7;
      width: 80px;
      height: 80px;
      background: #fff;
      text-align: center;
      border-radius: 49px;
      -webkit-border-radius: 49px;
      -moz-border-radius: 49px;
      -o-border-radius: 49px;
      line-height: 1.7em;
    }



    .market-update-left {
      padding: 0px;
    }
  </style>
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
  <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/js/bootstrap.min.js"></script>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">
</head>

<body>
  <div class="market-updates">
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-1">
        <div class="col-md-8 market-update-left">
          <h3>{{doctorName}}</h3>
          <h4>Doctor Name</h4>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-user-o"> </i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-2">
        <div class="col-md-8 market-update-left">
          <h3>{{symptoms}}</h3>
          <h4>Symptoms</h4>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-info-circle"></i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-3">
        <div class="col-md-8 market-update-left">
          <h3>{{doctorMobile}}</h3>
          <h4>Doctor Mobile</h4>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-mobile" aria-hidden="true"></i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="clearfix"> </div>
  </div>



  <br><br><br>



  <div class="market-updates">
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-1">
        <div class="col-md-8 market-update-left">
          <h3>{{doctorAddress}}</h3>
          <h4>Doctor Address</h4>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-map-marker"> </i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-2">
        <div class="col-md-8 market-update-left">
          <h3>{{doctorDepartment}}</h3>
          <h4>Doctor Department</h4>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-list-alt"></i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="col-md-4 market-update-gd">
      <div class="market-update-block clr-block-3">
        <div class="col-md-8 market-update-left">
          <h3>{{admitDate}}</h3>
          <h4>Admit Date</h4>
        </div>
        <div class="col-md-4 market-update-right">
          <i class="fa fa-calendar-o" aria-hidden="true"></i>
        </div>
        <div class="clearfix"> </div>
      </div>
    </div>
    <div class="clearfix"> </div>
  </div>

  <!--
    developed By : sumit kumar
    facebook : fb.com/sumit.luv
    youtube : youtube.com/lazycoders
    -->


</body>

</html>

{% extends 'hospital/patient_base.html' %}
{% load static %}
{% block content %}

<head>
  <meta charset="utf-8">
  <title>A simple, clean, and responsive HTML invoice template</title>

  <style>
    .invoice-box {
      max-width: 800px;
      margin: auto;
      padding: 30px;
      border: 1px solid #eee;
      box-shadow: 0 0 10px rgba(0, 0, 0, .15);
      font-size: 16px;
      line-height: 24px;
      font-family: 'Helvetica Neue', 'Helvetica', Helvetica, Arial, sans-serif;
      color: #555;
    }

    .invoice-box table {
      width: 100%;
      line-height: inherit;
      text-align: left;
    }

    .invoice-box table td {
      padding: 5px;
      vertical-align: top;
    }

    .invoice-box table tr td:nth-child(2) {
      text-align: right;
    }

    .invoice-box table tr.top table td {
      padding-bottom: 20px;
    }

    .invoice-box table tr.top table td.title {
      font-size: 45px;
      line-height: 45px;
      color: #333;
    }

    .invoice-box table tr.information table td {
      padding-bottom: 40px;
    }

    .invoice-box table tr.heading td {
      background: #eee;
      border-bottom: 1px solid #ddd;
      font-weight: bold;
    }

    .invoice-box table tr.details td {
      padding-bottom: 20px;
    }

    .invoice-box table tr.item td {
      border-bottom: 1px solid #eee;
    }

    .invoice-box table tr.item.last td {
      border-bottom: none;
    }

    .invoice-box table tr.total td:nth-child(2) {
      border-top: 2px solid #eee;
      font-weight: bold;
    }

    @media only screen and (max-width: 600px) {
      .invoice-box table tr.top table td {
        width: 100%;
        display: block;
        text-align: center;
      }

      .invoice-box table tr.information table td {
        width: 100%;
        display: block;
        text-align: center;
      }
    }

    /** RTL **/
    .rtl {
      direction: rtl;
      font-family: Tahoma, 'Helvetica Neue', 'Helvetica', Helvetica, Arial, sans-serif;
    }

    .rtl table {
      text-align: right;
    }

    .rtl table tr td:nth-child(2) {
      text-align: left;
    }

    .menu {
      top: 50px;
    }

    .download {
      text-align: center;
      display: block;
    }
  </style>
</head>

<br><br><br>

{%if is_discharged%}
<div class="invoice-box">
  <table cellpadding="0" cellspacing="0">
    <tr class="top">
      <td colspan="2">
        <table>
          <tr>
            <td class="title">
              <h1>Hospital Management</h1>
            </td>

            <td>

              Admit Date: {{admitDate}}<br>
              Release Date: {{releaseDate}}<br>
              Days Spent: {{daySpent}}
            </td>
          </tr>
        </table>
      </td>
    </tr>

    <tr class="information">
      <td colspan="2">
        <table>
          <tr>
            <td>
              Patient Name : {{patientName}}<br>
              Patient Mobile : {{mobile}}<br>
              Patient Addres : {{address}}<br>
            </td>

            <td>
              Doctor Name :<br>
              {{assignedDoctorName}}<br>

            </td>
          </tr>
        </table>
      </td>
    </tr>

    <tr class="heading">
      <td>
        Disease and Symptoms
      </td>
      <td>

      </td>

    </tr>

    <tr class="details">
      <td>
        {{symptoms}}
      </td>
    </tr>
    <tr class="heading">
      <td>
        Item
      </td>

      <td>
        Price
      </td>
    </tr>

    <tr class="item">
      <td>
        Room Charge of {{daySpent}} Days
      </td>

      <td>
        {{roomCharge}}
      </td>
    </tr>

    <tr class="item">
      <td>
        Doctor Fee
      </td>

      <td>
        {{doctorFee}}
      </td>
    </tr>

    <tr class="item">
      <td>
        Medicine Cost
      </td>

      <td>
        {{medicineCost}}
      </td>
    </tr>

    <tr class="item last">
      <td>
        Other Charge
      </td>

      <td>
        {{OtherCharge}}
      </td>
    </tr>

    <tr class="total">
      <td></td>

      <td>
        Total Rupees : {{total}}
      </td>
    </tr>

  </table>
</div>
<br><br>
<div class="download">
  <a style="background:red; width:500px;" href="{% url 'download-pdf' patientId  %}">Download</a>
</div>



{%else%}
<h5 style="text-align:center;">You Are Not Discharged By Hospital !</h5>
<h5 style="text-align:center;">Your Treatment Is Going On !</h5><br><br>
<h6 style="text-align:center;">When You Will Be Discahrged. You Can Download Invoice.</h6>
{%endif%}
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}

{% extends 'hospital/admin_base.html' %}
{% load static %}
{% block content %}

<head>
  <meta charset="utf-8">
  <title>A simple, clean, and responsive HTML invoice template</title>

  <style>
    .invoice-box {
      max-width: 800px;
      margin: auto;
      padding: 30px;
      border: 1px solid #eee;
      box-shadow: 0 0 10px rgba(0, 0, 0, .15);
      font-size: 16px;
      line-height: 24px;
      font-family: 'Helvetica Neue', 'Helvetica', Helvetica, Arial, sans-serif;
      color: #555;
    }

    .invoice-box table {
      width: 100%;
      line-height: inherit;
      text-align: left;
    }

    .invoice-box table td {
      padding: 5px;
      vertical-align: top;
    }

    .invoice-box table tr td:nth-child(2) {
      text-align: right;
    }

    .invoice-box table tr.top table td {
      padding-bottom: 20px;
    }

    .invoice-box table tr.top table td.title {
      font-size: 45px;
      line-height: 45px;
      color: #333;
    }

    .invoice-box table tr.information table td {
      padding-bottom: 40px;
    }

    .invoice-box table tr.heading td {
      background: #eee;
      border-bottom: 1px solid #ddd;
      font-weight: bold;
    }

    .invoice-box table tr.details td {
      padding-bottom: 20px;
    }

    .invoice-box table tr.item td {
      border-bottom: 1px solid #eee;
    }

    .invoice-box table tr.item.last td {
      border-bottom: none;
    }

    .invoice-box table tr.total td:nth-child(2) {
      border-top: 2px solid #eee;
      font-weight: bold;
    }

    @media only screen and (max-width: 600px) {
      .invoice-box table tr.top table td {
        width: 100%;
        display: block;
        text-align: center;
      }

      .invoice-box table tr.information table td {
        width: 100%;
        display: block;
        text-align: center;
      }
    }

    /** RTL **/
    .rtl {
      direction: rtl;
      font-family: Tahoma, 'Helvetica Neue', 'Helvetica', Helvetica, Arial, sans-serif;
    }

    .rtl table {
      text-align: right;
    }

    .rtl table tr td:nth-child(2) {
      text-align: left;
    }

    .menu {
      top: 50px;
    }

    .download {
      text-align: center;
      display: block;
    }
  </style>
</head>

<br><br><br>
<div class="invoice-box">
  <table cellpadding="0" cellspacing="0">
    <tr class="top">
      <td colspan="2">
        <table>
          <tr>
            <td class="title">
              <h1>Hospital Management</h1>
            </td>

            <td>

              Admit Date: {{admitDate}}<br>
              Release Date: {{todayDate}}<br>
              Days Spent: {{day}}
            </td>
          </tr>
        </table>
      </td>
    </tr>

    <tr class="information">
      <td colspan="2">
        <table>
          <tr>
            <td>
              Patient Name : {{name}}<br>
              Patient Mobile : {{mobile}}<br>
              Patient Addres : {{address}}<br>
            </td>

            <td>
              Doctor Name :<br>
              {{assignedDoctorName}}<br>

            </td>
          </tr>
        </table>
      </td>
    </tr>

    <tr class="heading">
      <td>
        Disease and Symptoms
      </td>
      <td>

      </td>

    </tr>

    <tr class="details">
      <td>
        {{symptoms}}
      </td>
    </tr>
    <tr class="heading">
      <td>
        Item
      </td>

      <td>
        Price
      </td>
    </tr>

    <tr class="item">
      <td>
        Room Charge of {{day}} Days
      </td>

      <td>
        {{roomCharge}}
      </td>
    </tr>

    <tr class="item">
      <td>
        Doctor Fee
      </td>

      <td>
        {{doctorFee}}
      </td>
    </tr>

    <tr class="item">
      <td>
        Medicine Cost
      </td>

      <td>
        {{medicineCost}}
      </td>
    </tr>

    <tr class="item last">
      <td>
        Other Charge
      </td>

      <td>
        {{OtherCharge}}
      </td>
    </tr>

    <tr class="total">
      <td></td>

      <td>
        Total Rupees : {{total}}
      </td>
    </tr>

  </table>
</div>
<br><br>
<div class="download">
  <a style="background:red; width:500px;" href="{% url 'download-pdf' patientId  %}">Download</a>
</div>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% endblock content %}


{% extends 'hospital/admin_base.html' %}
{% load static %}
{% block content %}

<head>
  <meta charset="utf-8">
  <title>A simple, clean, and responsive HTML invoice template</title>

  <style>
    .invoice-box {
      max-width: 800px;
      margin: auto;
      padding: 30px;
      border: 1px solid #eee;
      box-shadow: 0 0 10px rgba(0, 0, 0, .15);
      font-size: 16px;
      line-height: 24px;
      font-family: 'Helvetica Neue', 'Helvetica', Helvetica, Arial, sans-serif;
      color: #555;
    }

    .invoice-box table {
      width: 100%;
      line-height: inherit;
      text-align: left;
    }

    .invoice-box table td {
      padding: 5px;
      vertical-align: top;
    }

    .invoice-box table tr td:nth-child(2) {
      text-align: right;
    }

    .invoice-box table tr.top table td {
      padding-bottom: 20px;
    }

    .invoice-box table tr.top table td.title {
      font-size: 45px;
      line-height: 45px;
      color: #333;
    }

    .invoice-box table tr.information table td {
      padding-bottom: 40px;
    }

    .invoice-box table tr.heading td {
      background: #eee;
      border-bottom: 1px solid #ddd;
      font-weight: bold;
    }

    .invoice-box table tr.details td {
      padding-bottom: 20px;
    }

    .invoice-box table tr.item td {
      border-bottom: 1px solid #eee;
    }

    .invoice-box table tr.item.last td {
      border-bottom: none;
    }

    .invoice-box table tr.total td:nth-child(2) {
      border-top: 2px solid #eee;
      font-weight: bold;
    }

    @media only screen and (max-width: 600px) {
      .invoice-box table tr.top table td {
        width: 100%;
        display: block;
        text-align: center;
      }

      .invoice-box table tr.information table td {
        width: 100%;
        display: block;
        text-align: center;
      }
    }

    /** RTL **/
    .rtl {
      direction: rtl;
      font-family: Tahoma, 'Helvetica Neue', 'Helvetica', Helvetica, Arial, sans-serif;
    }

    .rtl table {
      text-align: right;
    }

    .rtl table tr td:nth-child(2) {
      text-align: left;
    }

    .menu {
      top: 50px;
    }
  </style>
</head>

<br><br><br>
<div class="invoice-box">
  <table cellpadding="0" cellspacing="0">
    <tr class="top">
      <td colspan="2">
        <table>
          <tr>
            <td class="title">
              <h1>Hospital Management</h1>
            </td>

            <td>

              Admit Date: {{admitDate}}<br>
              Release Date: {{todayDate}}<br>
              Days Spent: {{day}}
            </td>
          </tr>
        </table>
      </td>
    </tr>

    <tr class="information">
      <td colspan="2">
        <table>
          <tr>
            <td>
              Patient Name : {{name}}<br>
              Patient Mobile : {{mobile}}<br>
              Patient Addres : {{address}}<br>
            </td>

            <td>
              Doctor Name :<br>
              {{assignedDoctorName}}<br>

            </td>
          </tr>
        </table>
      </td>
    </tr>

    <tr class="heading">
      <td>
        Disease and Symptoms
      </td>
      <td>

      </td>

    </tr>

    <tr class="details">
      <td>
        {{symptoms}}
      </td>
    </tr>
    <tr class="heading">
      <td>
        Item
      </td>

      <td>
        Price
      </td>
    </tr>
    <form method="post">
      {% csrf_token %}

      <tr class="item">
        <td>
          Room Charge (Per Day)
        </td>

        <td>
          <input type="number" name="roomCharge" placeholder="In Rupees" value="">
        </td>
      </tr>

      <tr class="item">
        <td>
          Doctor Fee
        </td>

        <td>
          <input type="number" name="doctorFee" placeholder="In Rupees" value="">
        </td>
      </tr>

      <tr class="item">
        <td>
          Medicine Cost
        </td>

        <td>
          <input type="number" name="medicineCost" placeholder="In Rupees" value="">
        </td>
      </tr>

      <tr class="item last">
        <td>
          Other Charge
        </td>

        <td>
          <input type="number" name="OtherCharge" placeholder="In Rupees" value="">
        </td>
      </tr>

      <tr class="total">
        <td></td>

        <td>
          <input type="submit" name="submit" value="Generate Bill">
        </td>
      </tr>

    </form>
  </table>
</div>

<!--
   developed By : sumit kumar
   facebook : fb.com/sumit.luv
   youtube : youtube.com/lazycoders
   -->

{% endblock content %}

{% extends 'hospital/admin_base.html' %}
{% load static %}
{% block content %}

<head>
  <meta charset="utf-8">
  <title>A simple, clean, and responsive HTML invoice template</title>

  <style>
    .invoice-box {
      max-width: 800px;
      margin: auto;
      padding: 30px;
      border: 1px solid #eee;
      box-shadow: 0 0 10px rgba(0, 0, 0, .15);
      font-size: 16px;
      line-height: 24px;
      font-family: 'Helvetica Neue', 'Helvetica', Helvetica, Arial, sans-serif;
      color: #555;
    }

    .invoice-box table {
      width: 100%;
      line-height: inherit;
      text-align: left;
    }

    .invoice-box table td {
      padding: 5px;
      vertical-align: top;
    }

    .invoice-box table tr td:nth-child(2) {
      text-align: right;
    }

    .invoice-box table tr.top table td {
      padding-bottom: 20px;
    }

    .invoice-box table tr.top table td.title {
      font-size: 45px;
      line-height: 45px;
      color: #333;
    }

    .invoice-box table tr.information table td {
      padding-bottom: 40px;
    }

    .invoice-box table tr.heading td {
      background: #eee;
      border-bottom: 1px solid #ddd;
      font-weight: bold;
    }

    .invoice-box table tr.details td {
      padding-bottom: 20px;
    }

    .invoice-box table tr.item td {
      border-bottom: 1px solid #eee;
    }

    .invoice-box table tr.item.last td {
      border-bottom: none;
    }

    .invoice-box table tr.total td:nth-child(2) {
      border-top: 2px solid #eee;
      font-weight: bold;
    }

    @media only screen and (max-width: 600px) {
      .invoice-box table tr.top table td {
        width: 100%;
        display: block;
        text-align: center;
      }

      .invoice-box table tr.information table td {
        width: 100%;
        display: block;
        text-align: center;
      }
    }

    /** RTL **/
    .rtl {
      direction: rtl;
      font-family: Tahoma, 'Helvetica Neue', 'Helvetica', Helvetica, Arial, sans-serif;
    }

    .rtl table {
      text-align: right;
    }

    .rtl table tr td:nth-child(2) {
      text-align: left;
    }

    .menu {
      top: 50px;
    }
  </style>
</head>

<br><br><br>
<div class="invoice-box">
  <table cellpadding="0" cellspacing="0">
    <tr class="top">
      <td colspan="2">
        <table>
          <tr>
            <td class="title">
              <h1>Hospital Management</h1>
            </td>

            <td>

              Admit Date: {{admitDate}}<br>
              Release Date: {{todayDate}}<br>
              Days Spent: {{day}}
            </td>
          </tr>
        </table>
      </td>
    </tr>

    <tr class="information">
      <td colspan="2">
        <table>
          <tr>
            <td>
              Patient Name : {{name}}<br>
              Patient Mobile : {{mobile}}<br>
              Patient Addres : {{address}}<br>
            </td>

            <td>
              Doctor Name :<br>
              {{assignedDoctorName}}<br>

            </td>
          </tr>
        </table>
      </td>
    </tr>

    <tr class="heading">
      <td>
        Disease and Symptoms
      </td>
      <td>

      </td>

    </tr>

    <tr class="details">
      <td>
        {{symptoms}}
      </td>
    </tr>
    <tr class="heading">
      <td>
        Item
      </td>

      <td>
        Price
      </td>
    </tr>
    <form method="post">
      {% csrf_token %}

      <tr class="item">
        <td>
          Room Charge (Per Day)
        </td>

        <td>
          <input type="number" name="roomCharge" placeholder="In Rupees" value="">
        </td>
      </tr>

      <tr class="item">
        <td>
          Doctor Fee
        </td>

        <td>
          <input type="number" name="doctorFee" placeholder="In Rupees" value="">
        </td>
      </tr>

      <tr class="item">
        <td>
          Medicine Cost
        </td>

        <td>
          <input type="number" name="medicineCost" placeholder="In Rupees" value="">
        </td>
      </tr>

      <tr class="item last">
        <td>
          Other Charge
        </td>

        <td>
          <input type="number" name="OtherCharge" placeholder="In Rupees" value="">
        </td>
      </tr>

      <tr class="total">
        <td></td>

        <td>
          <input type="submit" name="submit" value="Generate Bill">
        </td>
      </tr>

    </form>
  </table>
</div>

<!--
   developed By : sumit kumar
   facebook : fb.com/sumit.luv
   youtube : youtube.com/lazycoders
   -->

{% endblock content %}



<!DOCTYPE html>

<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  <title>LazyCoder || sumit</title>


  <style media="screen">
    .jumbotron {
      margin-top: 0px;
      margin-bottom: 0px;
    }

    .jumbotron h1 {
      text-align: center;
    }

    .alert {
      margin: 0px;
    }
  </style>
</head>

<body>
  {% include "hospital/navbar.html" %}
  <br>
  <br>

  <div class="jumbotron" style="margin-top: 0px;
    margin-bottom: 0px;">
    <h1 class="display-4">Hello {{request.user.first_name}}</h1>
    <p class="lead">Your Account is not approved till now <br><br>Our Team is checking your profile <br><br> Soon your account will be confirmed !!!</p>
    <hr class="my-4">
    <p>Check Later</p>
    <p class="lead">
      <a class="btn btn-primary btn-lg" href="/logout" role="button">Logout For Now</a>
    </p>
  </div>

  {% include "hospital/footer.html" %}
</body>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>


{% extends 'hospital/homebase.html' %}
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

{% block content %}

<br>
<br>

<div class="jumbotron" style="margin-bottom:0px;">
  <h1 class="display-4" style="text-align:center;">Hello, Patient</h1>
  <p class="lead">Welcome to Hospital Management System.</p>
  <hr class="my-4">
  <p>You can access various features after Login/SignUp.</p>
  <p class="lead">
    <a class="btn btn-primary btn-lg" href="/patientsignup" role="button">Register Your Account</a>
    <a class="btn btn-primary btn-lg" href="/patientlogin" role="button">Login</a>
  </p>
</div>

{% endblock content %}
<!DOCTYPE html>
{% load widget_tweaks %}
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  <title>LazyCoder || sumit</title>


  <style type="text/css">
    body {
      color: #aa082e;
      background-color: #b6bde7;
      font-family: 'Roboto', sans-serif;
    }

    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }
  </style>

</head>

<body>
  {% include "hospital/navbar.html" %}
  <br>
  <br>
  <br><br>

  <!--- login page for patient by admin(sumit)  ---------->
  <form method="post">
    {% csrf_token %}
    <div class="container register-form">
      <div class="form">
        <div class="note">
          <p>Patient Login Page</p>
        </div>

        <div class="form-content">
          <div class="row">
            <div class="col-md-6">

              <div class="form-group">
                {% render_field form.username class="form-control" placeholder="Username" %}
              </div>

            </div>
            <div class="col-md-6">

              <div class="form-group">
                {% render_field form.password class="form-control" placeholder="Password" %}
              </div>

            </div>
          </div>
          <button type="submit" class="btnSubmit">Login</button>
          <div class="text-center">Do not have account? <a href="patientsignup">Signup here</a></div>
        </div>
      </div>
    </div>

  </form>

  <br><br><br>
  <!--
  developed By : sumit kumar
  facebook : fb.com/sumit.luv
  youtube : youtube.com/lazycoders
  -->

  {% include "hospital/footer.html" %}
</body>


<!DOCTYPE html>

{% load widget_tweaks %}
<html lang="en" dir="ltr">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  <title>LazyCoder || sumit</title>
  <style type="text/css">
    body {
      color: #aa082e;
      background-color: #b6bde7;
      font-family: 'Roboto', sans-serif;
    }

    a:link {
      text-decoration: none;
    }

    .note {
      text-align: center;
      height: 80px;
      background: -webkit-linear-gradient(left, #0072ff, #8811c5);
      color: #fff;
      font-weight: bold;
      line-height: 80px;
    }

    .form-content {
      padding: 5%;
      border: 1px solid #ced4da;
      margin-bottom: 2%;
    }

    .form-control {
      border-radius: 1.5rem;
    }

    .btnSubmit {
      border: none;
      border-radius: 1.5rem;
      padding: 1%;
      width: 20%;
      cursor: pointer;
      background: #0062cc;
      color: #fff;
    }
  </style>

</head>


<body>
  {% include "hospital/navbar.html" %}
  <br>
  <br>
  <br>
  <br>




  <!--- signup page for patient by admin(sumit)  ---------->
  <form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    <div class="container register-form">
      <div class="form">
        <div class="note">
          <p>Register to Hospital</p>
        </div>

        <div class="form-content">
          <div class="row">
            <div class="col-md-6">
              <div class="form-group">
                {% render_field userForm.first_name class="form-control" placeholder="First Name" %}
              </div>
              <div class="form-group">
                {% render_field userForm.username class="form-control" placeholder="Username" %}
              </div>
              <div class="form-group">
                {% render_field patientForm.address class="form-control" placeholder="Address" %}
              </div>
              <div class="form-group">
                {% render_field patientForm.symptoms class="form-control" placeholder="Symptoms" %}
              </div>
              <div class="form-group">
                {% render_field patientForm.profile_pic required="required" class="form-control" placeholder="Profile Picture" %}
              </div>
            </div>
            <div class="col-md-6">
              <div class="form-group">
                {% render_field userForm.last_name class="form-control" placeholder="Last Name" %}
              </div>
              <div class="form-group">
                {% render_field userForm.password class="form-control" placeholder="Password" %}
              </div>
              <div class="form-group">
                {% render_field patientForm.mobile class="form-control" pattern="[6789][0-9]{9}" placeholder="Mobile Number" %}
              </div>
              <div class="form-group">
                {% render_field patientForm.assignedDoctorId class="form-control" placeholder="Doctor" %}
              </div>

            </div>
          </div>
          <button type="submit" class="btnSubmit">Register</button>
          <div class="text-center">Already have an account? <a href="patientlogin">Login here</a></div>
        </div>
      </div>
    </div>

  </form>

  {% include "hospital/footer.html" %}
</body>
<!--
developed By : sumit kumar
facebook : fb.com/sumit.luv
youtube : youtube.com/lazycoders
-->

</html>


</html>



#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys


def main():
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'hospitalmanagement.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)


if __name__ == '__main__':
    main()







